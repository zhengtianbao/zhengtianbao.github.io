Zhengtianbao's Blog
===================

## 本地运行环境配置

### 1. 安装 ruby

注意：

- ubuntu 20.04 环境下用 apt 直接安装的 ruby 版本过低，需要通过 Rbenv 管理工具安装高版本
- 不要使用 zsh，用 bash 环境安装运行

安装依赖

```
sudo apt-get install ruby-full build-essential zlib1g-dev
```

安装 Rbenv

```
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc
```

安装 ruby-build

```
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

安装 ruby

```
rbenv install 3.1.2
rbenv global 3.1.2
```

安装 jekyll

```
gem install jekyll bundler
```

### 2. 运行

```
git clone https://github.com/zhengtianbao/zhengtianbao.github.io.git
```

安装依赖

```
bundle install
bundle
```

启动本地服务

```
bundle exec jekyll s
```

访问地址： http://127.0.0.1:4000
