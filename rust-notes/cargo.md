# Cargo工程上相关内容
* 通过release profile来自定义构建
* 在 https://crates.io 上发布库
* 通过workspaces组织大工程
* 从 https://crates.io 安装库
* 使用子自定义命令扩展cargo

## 通过release profile 来自定义构建
* release profile(发布配置)：
  - 是预定义的
  - 可自定义：可使用不同的配置，对代码编译拥有更多的控制
* 每个profile的配置都独立于其他的profile
* Cargo主要的两个profile：
  - dev profile：适用于开发，cargo build 
  - release profile: 适用于发布，cargo build --release

## 自定义profile
* 针对每个profile，Cargo都提供了默认的配置
* 如果想自定义xxxx profile的配置：
  - 可以在Cargo.toml里添加[profile.xxxx]区域，在里面覆盖默认配置的子集


