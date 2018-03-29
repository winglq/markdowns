title: K8S
author: qing
date: 2018-03-29
description: K8S
tags:
category:
acl: 00

# K8S代码分析

## 版本<a name="version"></a>

## Cobra
K8S使用Cobra做为命令行框架。

## init命令

### Features Gates

Feature gates are a set of key=value pairs that describe alpha or experimental features.

init命令会根据传入参数设置`SelfHosting`, `StoreCertsInSecrets`, `HighAvailability`, `CoreDNS`, `DynamicKubeletConfig`, `Auditing`这些feature.

Feature的数据结构

    type Feature struct {
    	utilfeature.FeatureSpec            //如下
    	MinimumVersion   *version.Version  //最小版本号，包括components,semver,PreRelease,buildMetadata. [K8S版本](#version)
    	HiddenInHelpText bool
    }

FeatureSpec的数据结构

    type FeatureSpec struct {
    	Default    bool
    	PreRelease prerelease
    }

