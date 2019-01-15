---
layout:     post
title:    轻松构建微服务之分布式事物
no-post-nav: true
category: other
tags: [arch]
excerpt: ACID MVCC
---


## 拜占庭将军问题

在存在消息丢失的不可信信道上通过消息传递的方式得到一致性是不可能的

由于当时拜占庭帝国国土辽阔，为了防御目的，每个军队都分隔很远，将军和将军之间只能靠信差来通信，但是军队内可能存在间谍和叛徒，那么如何在已知有叛徒的情况下，在其他忠诚的将军之间达成一致性协定。映射到网络问题，例如网络被劫持，网络不可达等


## CAP

## 2PC

## TCC

## FESCAR

## 一致性协议 raft

1.通过选举保证集群中只有一个leader,只有leader对外提供写服务，leader将日志广播给follower,每一条日志都按顺序维护在一个队列里，所有节点的队列里有一个index来控制前面的是已经提交的，后面的是没提交的，提交代表已经有超过半数的节点应答，leader先把日志复制给所有follower，在收到半数节点应答后在通知follower,index位置来控制那些日志是已经提交的，只有提交过的日志，follower才会提供给应用方使用

2、选举过程，当一个leader长时间没有应答时，所有的follower都可以成为candidate，向其他follower节点发送投票请求，如果超过半数follower节点应答后这个candidate就会升级为leader,为了避免所有的follower节点已经作为candidate发起投票，引入随机超时机制，每个follower和leader的超时时间在一定范围内随机，当candidate发起投票没有结果时，随机等待一定时间。

3.candidate的日志长度要大于等于半数follower节点的日志才能成为leader,所以发起投票的时候如果follower发现自己的日志长度大于后选择的就会投反对票

4.日志补齐，当leader发生故障的时候，各个follower上的状态不一样，所以新leader产生后需要补齐所有follow的日志，而且新leander的日志也不一定是最长的，但是foller日志上面没有的日志肯定是未提交的，这个时候补齐就可以

5.老leader复活，每一次选举到下一次选举的时间称为一个term任期，每一个任期内都会维护一个数字并自增，当leader发送复制请求的时候会带上term编号，如果follower发现term比自己小就拒绝，


raft设计中只有两个rpc请求，一个选举，一个复制日志，复制日志顺便做了心跳检测，当没有日志复制的时候发送空日志，


触发选举的唯一条件是 election timeout到期，每一个节点的 election timeout都会将自己设置为candidate然后发起投票，每个节点的election timeout都会存在一个随机值，所以不同，当一个节点被选为leader后会定期向所有的follower发送心跳包，follower收到心跳包后会延长election timeout的值。

节点选举的时候term值大的会优先于term值小的，每一轮选举term值都会加1
