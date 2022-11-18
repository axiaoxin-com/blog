---
title: "Golang 使用 Redis ZSET 实现排行榜"
author: "axiaoxin"
date: 2022-11-26T22:04:06+08:00
subtitle: ""
image: ""
tags: ["golang", "redis"]
slug: ""
---

本文介绍如何使用 Golang 采用 Redis 的有序集合 zset 实现一个用户排行榜。

用户排行榜按用户的某一种排序值进行排序，比如充值金额，当该排序值相同时，则需按达成该值的时间先后顺序进行排序，比如两个用户充值金额都是 100 元，则先充满 100 元的用户排名在前面。

Redis 的 zset 做这种排行榜是十分合适的，在不考虑时间先后顺序的情况下，只需将用户的排序值作为 zset 的 score， 用户 id 作为 zset 的 value，然后通过 zset 提供的方法进行查询即可。

当需要把时间维度也作为排序因素考虑时，我们可以将时间信息也放入到 score 中一起参与排序。

这里采用的方式是将 zset score 按十进制数拆分，整数部分用于表示用户排序值 val，小数部分表示排行活动结束时间戳（秒）与用户排序值更新时间戳（秒）的差值 deltaTs。

最好是将 score 十进制数字总长度限制为最长 16 位，超过 16 位的数会有浮点数精度导致发生进位，影响结果的正确性。小数部分使用时间戳差值 deltaTs 就是为了缩短整个数字的长度以保存更大的排序值。

小数部分的数字长度由 deltaTs 的数字长度确定，整数部分最大支持长度则为：`16-len(deltaTs)`。比如排行榜活动时长为 10 天，结束时间减去开始时间，总时间差为 864000 秒，长度为 6，则 deltaTs 宽度为 6，每当有用户更新排序值时，计算活动时间与当前时间的时间戳差值 deltaTs 作为小数部分，其最长为 6 位数，最小 1 位数，不够 6 位数时，则在前面补 0。当然，你需要根据你的实际情况进行修改，比如你需要使用毫秒维度。

这样整数部分为用户真实的排序值，小数部分为用户的时间信息，在排行活动结束时间固定的情况下，越早触发小数值（deltaTs）越大，就可以实现在整数部分的排序值相同时，按其时间进行排序。

原理说明完毕，现在来看看如何实现。

1、首先我们创建一个名叫 ZRanking 的结构体，并实现其 New 方法，redis 客户端使用[go-redis](github.com/go-redis/redis/)：

```golang
type ZRanking struct {
	Redis          *redis.Client
	Key            string        // redis zset key
	Expiration     time.Duration // 数据保存过期时间
	StartTimestamp int64         // 排行活动开始时间
	EndTimestamp   int64         // 排行活动结束时间戳
	TimePadWidth   int           // 排行榜活动结束时间与用户排序值更新时间的差值补0宽度
}


// New 创建ZRanking实例
func New(rds *redis.Client, key string, startTs, endTs int64, expiration time.Duration) (*ZRanking, error) {
	deltaTs := endTs - startTs
	if deltaTs <= 0 {
		return nil, fmt.Errorf("invalid deltaTs:%v", deltaTs)
	}
	timePadWidth := len(fmt.Sprint(deltaTs))
	return &ZRanking{
		Redis:          rds,
		Key:            key,
		Expiration:     expiration,
		StartTimestamp: startTs,
		EndTimestamp:   endTs,
		TimePadWidth:   timePadWidth,
	}, nil
}
```

在 New 方法中通过排行活动的结束时间戳和开始时间戳计算小数部分的最大长度 timePadWidth 用于补 0 处理。

2、当用户排序值 val 发生了变化，先将排序值 val 转换为 zset 中实际带有时间信息的 score：

```golang
// val 转为 score:
// score = float64(val.deltaTs)
func (r *ZRanking) val2score(ctx context.Context, val int64) (float64, error) {
	nowts := time.Now().Unix()
	deltaTs := r.EndTimestamp - nowts
	scoreFormat := fmt.Sprintf("%%v.%%0%dd", r.TimePadWidth)
	scoreStr := fmt.Sprintf(scoreFormat, val, deltaTs)
	score, err := strconv.ParseFloat(scoreStr, 64)
	if err != nil {
		err = errors.Wrap(err, "ZRanking val2score ParseFloat error")
		return 0, err
	}
	return score, nil
}


// 从 score 中获取 val
func (r *ZRanking) score2val(ctx context.Context, score float64) (int64, error) {
	scoreStr := fmt.Sprint(score)
	ss := strings.Split(scoreStr, ".")
	valStr := ss[0]
	val, err := strconv.ParseInt(valStr, 10, 64)
	if err != nil {
		err = errors.Wrap(err, "ZRanking score2val ParseInt error")
		return 0, err
	}
	return val, nil
}
```

在进行小数处理时，需要进行 scoreFormat 的补 0 处理。将 score 转换为排序值 val 就比较简单了，直接按小数点分割取整数部分即可。

3、更新 zset：先取出用户之前的 score，去除小数部分得到整数部分的排序值 val，加上上一步中转换得到的 score 即为用户的最新 score，再将其写入 redis。由于需要先查然后计算最后才写，为了保证结果的正确这里采用 redis lua 实现这段逻辑：

```lua
-- 排行榜 key
local key = KEYS[1]
-- 要更新的用户 ID
local uid = ARGV[1]
-- 用户本次新增的 val （小数位为时间差标识）
local valScore = ARGV[2]
-- 获取用户之前的 score
local score = redis.call("ZSCORE", key, uid)
if score == false then
    score = 0
end
-- 从 score 中抹除用于时间差标识的小数部分，获取整数的排序 val
local val = math.floor(score)
-- 更新用户最新的 score 信息（累计 val.最新时间差）
local newScore = valScore+val
redis.call("ZADD", key, newScore, uid)
-- 更新成功返回 newScore（注意要使用 tostring 才能返回小数）
return tostring(newScore)
```

4、排行榜查询：

通过 go-redis 的 ZRangeWithScores 或 ZRevRangeWithScores 即可查询整个排行榜，查询出的 score 通过上面 score2val 方法获取 score 中的排序值。

通过 ZRank 或 ZRevRank 即可查询某个用户的排行。

通过 ZScore + score2val 即可查询某个用户 score 中的排序值。

通过 ZCard 即可查询排行榜的总人数。

最后，附上该排行榜的 Golang 实现封装 [zranking](https://github.com/axiaoxin-com/zranking) ，符合需求的可以直接使用，使用示例可参考 [exmaple](https://github.com/axiaoxin-com/zranking/blob/main/_example/main.go) 。
