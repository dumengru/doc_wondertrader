# IUftStraCtx.h

source: `{{ page.path }}`

```cpp
/*!
 * \file IUftStraCtx.h
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#pragma once
#include <stdint.h>
#include <string>
#include "ExecuteDefs.h"

#include "../Includes/WTSMarcos.h"

NS_WTP_BEGIN
class WTSCommodityInfo;
class WTSTickSlice;
class WTSKlineSlice;
class WTSTickData;
struct WTSBarStruct;

// UFT策略上下文管理器接口
class IUftStraCtx
{
public:
	IUftStraCtx(const char* name) :_name(name) {}
	virtual ~IUftStraCtx() {}

	const char* name() const { return _name.c_str(); }

public:
	virtual uint32_t id() = 0;

	//回调函数
	virtual void on_init() = 0;
	virtual void on_tick(const char* stdCode, WTSTickData* newTick) = 0;
	virtual void on_order_queue(const char* stdCode, WTSOrdQueData* newOrdQue) = 0;
	virtual void on_order_detail(const char* stdCode, WTSOrdDtlData* newOrdDtl) = 0;
	virtual void on_transaction(const char* stdCode, WTSTransData* newTrans) = 0;
	virtual void on_bar(const char* stdCode, const char* period, uint32_t times, WTSBarStruct* newBar) {}

	/*
	 *	回测结束事件
	 *	只在回测下才会触发
	 */
	virtual void	on_bactest_end() {};

	virtual void	on_tick_updated(const char* stdCode, WTSTickData* newTick) {}
	virtual void	on_ordque_updated(const char* stdCode, WTSOrdQueData* newOrdQue) {}
	virtual void	on_orddtl_updated(const char* stdCode, WTSOrdDtlData* newOrdDtl) {}
	virtual void	on_trans_updated(const char* stdCode, WTSTransData* newTrans) {}

	//策略接口
	/*
	 *	撤单接口
	 *	@localid	本地单号
	 */
	virtual bool		stra_cancel(uint32_t localid) = 0;

	/*
	 *	开多
	 *	@stdCode	代码，格式如SSE.600000
	 *	@price		委托价格
	 *	@qty		下单数量
	 */
	virtual uint32_t	stra_enter_long(const char* stdCode, double price, double qty) { return 0; }

	/*
	 *	开空
	 *	@stdCode	代码，格式如SSE.600000
	 *	@price		委托价格
	 *	@qty		下单数量
	 */
	virtual uint32_t	stra_enter_short(const char* stdCode, double price, double qty) { return 0; }

	/*
	 *	平多
	 *	@stdCode	代码，格式如SSE.600000
	 *	@price		委托价格
	 *	@qty		下单数量
	 *	@isToday	是否今仓，SHFE、INE专用
	 */
	virtual uint32_t	stra_exit_long(const char* stdCode, double price, double qty, bool isToday = false) { return 0; }

	/*
	 *	平空
	 *	@stdCode	代码，格式如SSE.600000
	 *	@price		委托价格
	 *	@qty		下单数量
	 *	@isToday	是否今仓，SHFE、INE专用
	 */
	virtual uint32_t	stra_exit_short(const char* stdCode, double price, double qty, bool isToday = false) { return 0; }

	/*
	 *	获取品种信息
	 *	@stdCode	代码，格式如SSE.600000
	 */
	virtual WTSCommodityInfo* stra_get_comminfo(const char* stdCode) = 0;

	/*
	 *	获取K线，暂未实现
	 *	@stdCode	代码，格式如SSE.600000
	 *	@period		周期，如m1/m5/d1
	 *	@count		条数
	 */
	virtual WTSKlineSlice*	stra_get_bars(const char* stdCode, const char* period, uint32_t count) = 0;

	/*
	 *	获取tick，暂未实现
	 *	@stdCode	代码，格式如SSE.600000
	 *	@count		条数
	 */
	virtual WTSTickSlice*	stra_get_ticks(const char* stdCode, uint32_t count) = 0;

	/*
	 *	获取委托明细，暂未实现
	 *	@stdCode	代码，格式如SSE.600000
	 *	@count		条数
	 */
	virtual WTSOrdDtlSlice*	stra_get_order_detail(const char* stdCode, uint32_t count) = 0;

	/*
	 *	获取委托队列，暂未实现
	 *	@stdCode	代码，格式如SSE.600000
	 *	@count		条数
	 */
	virtual WTSOrdQueSlice*	stra_get_order_queue(const char* stdCode, uint32_t count) = 0;

	/*
	 *	获取逐笔成交，暂未实现
	 *	@stdCode	代码，格式如SSE.600000
	 *	@count		条数
	 */
	virtual WTSTransSlice*	stra_get_transaction(const char* stdCode, uint32_t count) = 0;

	/*
	 *	读取最后一笔tick
	 *	@stdCode	代码，格式如SSE.600000
	 */
	virtual WTSTickData*	stra_get_last_tick(const char* stdCode) = 0;

	/*
	 *	获取持仓
	 *	@stdCode	代码，格式如SSE.600000
	 *	@bOnlyValid	获取可用持仓
	 *	@iFlag		读取标记，1-多头，2-空头，3-净头寸
	 */
	virtual double stra_get_position(const char* stdCode, bool bOnlyValid = false, int32_t iFlag = 3) = 0;

	/*
	 *	获取最新价格
	 *	@stdCode	代码，格式如SSE.600000
	 */
	virtual double stra_get_price(const char* stdCode) = 0;

	/*
	 *	获取未完成手数，买入为正，卖出为负
	 *	@stdCode	代码，格式如SSE.600000
	 */
	virtual double stra_get_undone(const char* stdCode) = 0;

	/*
	 *	订阅接口
	 */
	virtual void stra_sub_ticks(const char* stdCode) = 0;
	virtual void stra_sub_order_queues(const char* stdCode) = 0;
	virtual void stra_sub_order_details(const char* stdCode) = 0;
	virtual void stra_sub_transactions(const char* stdCode) = 0;

	/*
	 *	输出日志
	 */
	virtual void stra_log_info(const char* message) = 0;
	virtual void stra_log_debug(const char* message) = 0;
	virtual void stra_log_error(const char* message) = 0;

protected:
	std::string _name;
};

NS_WTP_END
```