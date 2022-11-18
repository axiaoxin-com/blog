---
title: "Python 有序更新 yaml 文件"
author: "axiaoxin"
date: 2015-05-17T11:03:07+08:00
subtitle: ""
image: ""
tags: ["python"]
slug: ""
---

在 Python 中使用 PyYAML 操作 yml 文件，yaml 默认 load 进来是个字典，所以无法保持原本的顺序，要想不改变原本的 yaml 结构，更新 yaml 文件内容需要用到以下方法

```python
from collections import OrderedDict
import yaml

def ordered_yaml_load(yaml_path, Loader=yaml.Loader,
                    object_pairs_hook=OrderedDict):
    class OrderedLoader(Loader):
        pass

    def construct_mapping(loader, node):
        loader.flatten_mapping(node)
        return object_pairs_hook(loader.construct_pairs(node))
    OrderedLoader.add_constructor(
        yaml.resolver.BaseResolver.DEFAULT_MAPPING_TAG,
        construct_mapping)
    with open(yaml_path) as stream:
        return yaml.load(stream, OrderedLoader)


def ordered_yaml_dump(data, stream=None, Dumper=yaml.SafeDumper, **kwds):
    class OrderedDumper(Dumper):
        pass

    def _dict_representer(dumper, data):
        return dumper.represent_mapping(
            yaml.resolver.BaseResolver.DEFAULT_MAPPING_TAG,
            data.items())
    OrderedDumper.add_representer(OrderedDict, _dict_representer)
    return yaml.dump(data, stream, OrderedDumper, **kwds)

# load yaml file
yaml_dict = ordered_yaml_load(yaml_file_path)

# dump new yaml file
with open(yaml_file_path, 'w') as f:
    ordered_yaml_dump(new_yaml_dict, f, default_flow_style=False)
```
