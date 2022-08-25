# Ethash算法在线校验

## 本地部署

``` bash
# 安装依赖
npm install

# 建立本地开发环境
npm run dev

# 构建最终静态文件
npm run build
```
## 功能介绍
1. 根据区块号在线获取区块数据
2. 计算区块RLP编码，计算区块封装哈希
3. 生成Ethash校验缓存，校验区块是否满足难度要求
