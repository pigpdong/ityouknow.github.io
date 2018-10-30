---
layout:     post
title:    互金公司怎么扣产品份额
no-post-nav: true
category: other
tags: [arch]
excerpt: 秒杀扣份额
---


## 扣份额

扣份额在互金行业是个很严肃的话题，产品的超卖和少卖都是事故，产品超卖了，我们收了投资人的钱但是没办法给他撮合借款人，产品少卖了，借款人那边就无法放款，总之体验差，严重情况下还会造成资损，而且扣份额一旦有bug，那将导致大批量的产品产生超卖或者少卖，而P2P行业是和资金打交道的，这无疑会是一个严重的事故。
所以产品的扣份额接口要幂等，要保证强一致性，同时，互金的产品和百货电商相比，种类少，有些公司可能一天就发一两个产品，偶会还会定时开售，这无疑会造成在同一时刻大量的投资人下单，有点类似于秒杀和抢购的场景，那么我们如何能保份额系统在流量洪峰下能够保持一致性和可用性？

### 先扣份额还是先支付？
先扣份额在支付，这种利用数据库事物机制控制商品库存，是最简单最精准的方式，这种方式绝对不会出现商品超卖的情况，但是如果一大批的用户卡里余额不足，将会导致这部分用户占用了库存导致其他用户想买的时候没有份额，如果支付公司接口返回的很快的话，将进行份额回滚，如果长时间才返回结果的话，这个时间的影响就需要考虑下了，我们这里暂时不关心某些友商可能恶意下单不付款的情况，如果支付公司支付接口返回处理中，我们此时是回滚份额还是继续占有呢？如果继续占有的话，支付公司一直不返回结果的话，那么这个产品将一直无法售罄，可能就无法放款给借款人，所以互金公司一般会选择份额回滚，或者为其等待10分钟，如果10分钟之内还是没有明确结果，在进行份额回滚，如果之后支付公司返回扣款成功，我们应该尝试重新去扣份额，如果此时产品已经售罄，将走退款流程给用户，如果重新扣份额成功，我们可以继续后面的流程，这里重新扣一次份额，主要是扣款成功了而用户没有购买成功用户体验很差，并且如果是一个很大的产品确实可能还存在多余份额没有卖完，那么如果需要重新扣一次份额，就需要扣份额的接口能够支持扣多次份额，为了保证幂等，同一笔单号需要允许扣多次份额，这个点我们后面在分析

先支付在扣份额，这种如果并发量很高的情况下，可能导致很多人下单并且付款成功，而去扣份额的时候发现没有库存，互金行业和传统电商不同，没办法让商家能否加库存，所以只能给用户退款，这在用户体验和成本（退款）上都很难容忍，

综上，互金公司更多地采用，先扣份额，在付款，如果付款结果处理中，可以等待一定时间后如果还是没有返回明确结果就回滚份额，等后续支付结果成功后，重新扣份额，如果扣份额失败就走退款流程，在实现上，我们要保证扣份额接口的幂等性，就是一个单号允许多次扣份额，这个保证了一致性，同时为了保证可用性，我们可能需要采取一些手段来让扣份额接口快速返回，最好是不加锁。

![](https://pigpdong.github.io/assets/images/2018/sharechange/sharechange.png)

### 怎么扣？

上文提到，扣份额要保证幂等，并且要支持一个单号能够多次扣份额,所以我们最好是在扣份额的时候，生成一笔份额变动流水，这个流水需要和订单号关联，而份额回滚也对应一笔流水，这样下次同一笔单号过来扣份额的时候就可以知道有没有扣过份额了，但是为了保证一致性，我们需要将这个流水变动和修改份额作为一个数据库本地事物提交。

```
@Override
public Result<Product> buckle(final ProdShareChangeReq req) {
// 1. 幂等性支持, 判断是否扣除过份额, 扣除过->直接返回
		Pair<Boolean, ProdShareChangeFlow> pairResult = shareChangeFlowService.checkAndGetBuckleResult(req.getSource(), req.getSeqId());
		final ProdShareChangeFlow flow = pairResult.getValue1();
		if (pairResult.getValue0() == true) {
			logger.info("buckle end >> SUCC, userId:{}, productId:{}, orderId:{}, useTime:{}ms, remark:{}",
					new Object[] { req.getUserId(), req.getProductId(), req.getOrderId(), watch.getTime(), "buckle has executed" });
			// 如果是预约购买扣份额, 更新订单ID到流水中
			setOrderIdToChangeFlow(flow, req.getOrderId());
			result.setSuccess(true);
			result.setData(cacheProd);
			return result;
		}

		// 2. 是否可以购买业务校验
		boolean canBuckle = isCanBuckle(req, cacheProd, result);
		if (canBuckle == false) {
			return result;
		}
		// 3. 执行扣份额操作
		Pair<Integer, String> pair = transactionTemplate.execute(new TransactionCallback<Pair<Integer, String>>() {
			@Override
			public Pair<Integer, String> doInTransaction(TransactionStatus status) {
				return execBuckle(req, flow);
			}
		});

}

public Pair<Integer, String> execBuckle(ProdShareChangeReq req, ProdShareChangeFlow flow) {
		int source = req.getSource();
		String seqId = req.getSeqId();
		Product cacheProd = productQueryService.getProductById(req.getProductId());
		// 1. 扣份额处理流程, 分预约扣份额和正常购买扣份额, 预约购买时不扣份额, 再第一步加锁校验时返回
		int result = -1;
		if (source == ShareChangeFlowSourceConstants.PRE_SHARE) {
			result = productDAO.decreasePreRemainAmountById(req.getProductId(), req.getAmount());
		} else if (source == ShareChangeFlowSourceConstants.NORMAL_PURCHASE || source == ShareChangeFlowSourceConstants.BATCH_PURCHASE) {
			result = productDAO.decreaseRemainAmountById(req.getProductId(), req.getAmount());
		} else {
			return new Pair<Integer, String>(ShareChangeFlowStatusConstants.NOT_BUCKLE, "扣份额失败,不要闹,这个交易类型不支持!");
		}
		/********* 扣份额成功 **********/
		if (result > 0) {
			// 份额流水不存在->新增, 存在->修改状态为成功, 并发时插入不进去
			if (flow == null) {
				addShareChangeFlow(req, cacheProd.getName(), ShareChangeFlowStatusConstants.BUCKLE_SUCC);
			} else {
				ProdShareChangeFlow lockFlow = shareChangeFlowService.selectBySourceAndOrderIdForUpdate(source, seqId);
				if (lockFlow.getStatus() == ShareChangeFlowStatusConstants.BUCKLE_SUCC) {
					throw new RuntimeException("亲, 您已经扣过份额了");
				}
				updateChangeFlowStatus(flow.getId(), ShareChangeFlowStatusConstants.BUCKLE_SUCC);
			}
			return new Pair<Integer, String>(ShareChangeFlowStatusConstants.BUCKLE_SUCC, "扣份额成功啦!");
			/******** 扣份额失败 **********/
		} else {
			return new Pair<Integer, String>(ShareChangeFlowStatusConstants.NOT_BUCKLE, "扣份额失败!");
		}
	}

```

上面这种方式在份额变动流水已经存在的情况下,还是会利用mysql的悲观锁,select for update ,因为如果后面进行回滚如果持续并发还是会出现数据不一致

以下为扣份额的sql ,减少剩余金额，增加销售金额，并且判断剩余金额大于本次扣除的金额，这样可以通过where条件的限制避免使用mysql的悲观锁，数据库可以增加检验，份额字段不允许小于0

```
UPDATE		prod_product
SET		remain_amount = remain_amount - #{buyAmount},
        trade_real_num = trade_real_num + 1,
        trade_num = trade_num + 1,
        trade_real_amount = trade_real_amount + #{buyAmount},
        trade_amount =		trade_amount + #{buyAmount},
        last_update_time = now()
WHERE		id = #{id}
and remain_amount >= #{buyAmount}

```

当然,为了保证数据的绝对准确,最好每天做对账,将所有的订单和份额流水进行关联对账,来发现是否存在状态和金额不一致的情况,实时证明这个步骤还是很必要的,因为没有人能够保证程序完全没有漏洞,而对账可以让我们能够尽早发现问题

如何应对抢购的情况，针对比较热的产品，我们可以将库存在redis中，针对特别热的秒杀情况，我们甚至可以先预热缓存到本地，但是可以将金额稍微在各机器上均分下，这样可以减少部分流量到服务器，反正服务器那边会重新校验份额的，如果量还是很大的话，在最后扣库存的时候可能会有大量的线程去竞争mysql innodb的行锁，这个也会影响数据库的吞吐量，这个就是单点热点商品可能会影响整个数据库的性能，如果我们公司的产品有一天确实火到这种程度，我们可以从以下两个方案进行解决

1. 应用层做排队 通过一个队列对下单购买的人做排队，对于完全无法处理的订单可以舍弃，这样减少了数据库的并发度，应用层只能做单机的排队，除非放到分布式缓存中
2. 数据库层做排队，数据库层做全局的排队，这种需要在innodb上开发补丁包，淘宝有这种实现并且已经开源

