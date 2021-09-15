#!/usr/bin/env bash

baseDir="/Users/robin/docker/ipfs/nodes"

nodeName=""

function getNodeName() {
    nodeName="ipfs-node-$1"
}

# 初始化 swarm key
function initSwarmKey {
    echo "初始化 swarm key"
    go get -u github.com/Kubuxu/go-ipfs-swarm-key-gen/ipfs-swarm-key-gen
    ~/go/bin/ipfs-swarm-key-gen > swarm.key
}

# 初始化所有节点
function initNodes {
    nodeCount=$1
    index=0
    while((${index}<nodeCount))
    do
        createNode ${index}

        let "index++"
    done
}

# 创建节点
function createNode {
    echo
    index=$1
    echo "初始化 node ${index} 目录结构"
    nodeDir="${baseDir}/node${index}"
    if [[ -e ${nodeDir} ]]
    then
        rm -rf ${nodeDir}
        echo "删除 ${nodeDir}"
    fi
    mkdir ${nodeDir}

    dataDir="${nodeDir}/data"
    exportDir="${nodeDir}/export"

    mkdir ${dataDir}
    mkdir ${exportDir}

    cp swarm.key ${dataDir}/swarm.key

    getNodeName ${index}
    echo "创建节点 ${nodeName}"

    docker rm -f ${nodeName}

    echo "创建节点容器，并启动 daemon"
    docker run \
        --name ${nodeName} \
        -v ${exportDir}:/export \
        -v ${dataDir}:/data/ipfs \
        -p $((10000 + index)):4001 \
        -p $((11000 + index)):5001 \
        -p $((12000 + index)):8080 \
        -d ipfs/go-ipfs:latest
}

# 清理公网引导节点
function clearBootstraps() {
    echo "10s 后开始清理公链引导节点"
    index=0
    while((${index}<10))
    do
        printf ". "
        sleep 1
        let "index++"
    done
    echo

    echo "开始清空公链引导节点"
    nodeCount=$1
    index=0
    while((${index}<nodeCount))
    do
        getNodeName ${index}
        echo "清理 ${nodeName} 引导节点"
        docker exec ${nodeName} ipfs bootstrap rm --all
        let "index++"
    done
}

function main() {
    nodeCount=10
    if [[ -e ${baseDir} ]]
    then
        echo "目录已存在"
        # rm -rf ${baseDir}
        # echo "删除 ${baseDir}"
    else
        mkdir ${baseDir}
    fi
    echo "创建 ${nodeCount} 个节点"
    initSwarmKey
    initNodes ${nodeCount}
    clearBootstraps ${nodeCount}

    echo "完成"
}
main
