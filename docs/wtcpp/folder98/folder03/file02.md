# WTSDataFactory.h

source: `{{ page.path }}`

```cpp
/*!
 * \file WTSDataFactory.h
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 数据拼接工厂类定义
 */
#pragma once
#include "../Includes/IDataFactory.h"

USING_NS_WTP;

// 数据拼接工厂实现(继承自数据工厂接口)
class WTSDataFactory : public IDataFactory
{
public:
	/*
	 *	利用tick数据更新K线
	 *	@klineData	K线数据
	 *	@tick		tick数据
	 *	@sInfo		交易时间模板
	 */
	virtual WTSBarStruct*	updateKlineData(WTSKlineData* klineData, WTSTickData* tick, WTSSessionInfo* sInfo);

	/*
	 *	利用基础周期K线数据更新K线
	 *	@klineData		K线数据
	 *	@newBasicBar	基础周期K线数据
	 *	@sInfo			交易时间模板
	 */
	virtual WTSBarStruct*	updateKlineData(WTSKlineData* klineData, WTSBarStruct* newBasicBar, WTSSessionInfo* sInfo);

	/*
	 *	从基础周期K线数据提取非基础周期的K线数据
	 *	@baseKline	基础周期K线
	 *	@period		基础周期，m1/m5/day
	 *	@times		周期倍数
	 *	@sInfo		交易时间模板
	 *	@bIncludeOpen	是否包含未闭合的K线
	 */
	virtual WTSKlineData*	extractKlineData(WTSKlineSlice* baseKline, WTSKlinePeriod period, uint32_t times, WTSSessionInfo* sInfo, bool bIncludeOpen = true);

	/*
	 *	从tick数据提取秒周期的K线数据
	 *	@ayTicks	tick数据
	 *	@seconds	目标周期
	 *	@sInfo		交易时间模板
	 *	@bUnixTime	tick时间戳是否是unixtime
	 */
	virtual WTSKlineData*	extractKlineData(WTSTickSlice* ayTicks, uint32_t seconds, WTSSessionInfo* sInfo, bool bUnixTime = false);

	/*
	 *	合并K线
	 *	@klineData	目标K线
	 *	@newKline	待合并的K线
	 */
	virtual bool			mergeKlineData(WTSKlineData* klineData, WTSKlineData* newKline);

protected:
	WTSBarStruct* updateMin1Data(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSTickData* tick);
	WTSBarStruct* updateMin5Data(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSTickData* tick);
	WTSBarStruct* updateDayData(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSTickData* tick);
	WTSBarStruct* updateSecData(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSTickData* tick);

	WTSBarStruct* updateMin1Data(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSBarStruct* newBasicBar);
	WTSBarStruct* updateMin5Data(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSBarStruct* newBasicBar);

	WTSKlineData* extractMin1Data(WTSKlineSlice* baseKline, uint32_t times, WTSSessionInfo* sInfo, bool bIncludeOpen = true);
	WTSKlineData* extractMin5Data(WTSKlineSlice* baseKline, uint32_t times, WTSSessionInfo* sInfo, bool bIncludeOpen = true);
	WTSKlineData* extractDayData(WTSKlineSlice* baseKline, uint32_t times, bool bIncludeOpen = true);

protected:
	static uint32_t getPrevMinute(uint32_t curMinute, int period = 1);
};
```

## WTSDataFactory.cpp

```cpp
/*!
 * \file WTSDataFactory.cpp
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#include "WTSDataFactory.h"
#include "../Includes/WTSDataDef.hpp"
#include "../Includes/WTSContractInfo.hpp"
#include "../Includes/WTSSessionInfo.hpp"
#include "../Share/TimeUtils.hpp"

using namespace std;

// 根据K线周期使用tick数据更新k线
WTSBarStruct* WTSDataFactory::updateKlineData(WTSKlineData* klineData, WTSTickData* tick, WTSSessionInfo* sInfo)
{
	if(klineData == NULL || tick == NULL)
		return NULL;

	if(strcmp(klineData->code(), tick->code()) != 0)
		return NULL;

	if(sInfo == NULL)
		return NULL;

	if (!sInfo->isInTradingTime(tick->actiontime() / 100000))
		return NULL;

	WTSKlinePeriod period = klineData->period();
	switch( period )
	{
	case KP_Tick:
		return updateSecData(sInfo, klineData, tick);
		break;
	case KP_Minute1:
		return updateMin1Data(sInfo, klineData, tick);
	case KP_Minute5:
		return updateMin5Data(sInfo, klineData, tick);
	case KP_DAY:
		return updateDayData(sInfo, klineData, tick);
	default:
		return NULL;
	}
}

// 根据K线周期, 使用bar更新K线
WTSBarStruct* WTSDataFactory::updateKlineData(WTSKlineData* klineData, WTSBarStruct* newBasicBar, WTSSessionInfo* sInfo)
{
	if (klineData == NULL || newBasicBar == NULL)
		return NULL;

	if (sInfo == NULL)
		return NULL;

	WTSKlinePeriod period = klineData->period();
	switch (period)
	{
	case KP_Minute1:
		return updateMin1Data(sInfo, klineData, newBasicBar);
	case KP_Minute5:
		return updateMin5Data(sInfo, klineData, newBasicBar);
	default:
		return NULL;
	}
}

// 用bar数据更新1分钟的K线
WTSBarStruct* WTSDataFactory::updateMin1Data(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSBarStruct* newBasicBar)
{
	if (sInfo == NULL)
		return NULL;

	if(klineData->times() == 1)
	{
		klineData->appendBar(*newBasicBar);
		klineData->setClosed(true);
		return klineData->at(-1);
	}

	//计算时间步长
	uint32_t steplen = klineData->times();

	const WTSBarStruct& curBar = *newBasicBar;

	uint32_t uTradingDate = curBar.date;
	uint32_t uDate = TimeUtils::minBarToDate(curBar.time);
	if (uDate == 19900000)
		uDate = uTradingDate;
	uint32_t uTime = TimeUtils::minBarToTime(curBar.time);
	uint32_t uMinute = sInfo->timeToMinutes(uTime) - 1;

	uint32_t uBarMin = (uMinute / steplen)*steplen + steplen;
	uint64_t uBarTime = sInfo->minuteToTime(uBarMin);
	//if(uBarTime > uTime && !sInfo->isInAuctionTime(uTime))
	//{
	//	//这种情况只可能是日期倒退
	//	uDate = TimeUtils::getNextDate(uDate, -1);
	//}
	uBarTime = TimeUtils::timeToMinBar(uDate, (uint32_t)uBarTime);

	WTSBarStruct* lastBar = NULL;
	if (klineData->size() > 0)
	{
		lastBar = klineData->at(klineData->size() - 1);
	}

	bool bNewBar = false;
	if (lastBar == NULL || lastBar->date != uDate || lastBar->time != uBarTime)
	{
		//只要日期和时间都不符,则认为已经是一条新的bar了
		lastBar = new WTSBarStruct();
		bNewBar = true;

		memcpy(lastBar, &curBar, sizeof(WTSBarStruct));
		lastBar->date = uDate;
		lastBar->time = uBarTime;
	}
	else
	{
		bNewBar = false;

		lastBar->high = max(lastBar->high, curBar.high);
		lastBar->low = min(lastBar->low, curBar.low);
		lastBar->close = curBar.close;
		lastBar->settle = curBar.settle;

		lastBar->vol += curBar.vol;
		lastBar->money += curBar.money;
		lastBar->add += curBar.add;
		lastBar->hold = curBar.hold;
	}

	if(lastBar->time > curBar.time)
	{
		klineData->setClosed(false);
	}
	else
	{
		klineData->setClosed(true);
	}

	if (bNewBar)
	{
		klineData->appendBar(*lastBar);
		delete lastBar;

		return klineData->at(-1);
	}

	return NULL;
}

// tick数据更新1分钟的K线
WTSBarStruct* WTSDataFactory::updateMin1Data(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSTickData* tick)
{
	uint32_t steplen = klineData->times();

	uint32_t uDate = tick->actiondate();				// 数据发生日期
	uint32_t uTime = tick->actiontime() / 100000;		// 数据发生时间 1059
	uint32_t uMinute = sInfo->timeToMinutes(uTime);		// 将时间转为分钟数
	if(uMinute == INVALID_UINT32)
	{
		// 如果时间处理出错, 直接用传入的 tick 更新 klineData 最后一条数据
		if(tick->volume() != 0)
		{
			WTSBarStruct *bar = klineData->at(klineData->size()-1);
			bar->close = tick->price();
			bar->high = max(bar->high,tick->price());
			bar->low = min(bar->low,tick->price());
			bar->vol += tick->volume();
			bar->money += tick->turnover();
			bar->hold = tick->openinterest();
			bar->add += tick->additional();
		}

		return NULL;
	}
	if (sInfo->isLastOfSection(uTime))								// 如果是收盘时间
	{
		uMinute--;
	}
	uint32_t uBarMin = (uMinute/steplen)*steplen + steplen;			// 时间+1步长
	uint32_t uOnlyMin = sInfo->minuteToTime(uBarMin);				// 将分钟数变为时间
	if(uOnlyMin == 0)
	{
		uDate = TimeUtils::getNextDate(uDate);
	}
	uint64_t uBarTime = TimeUtils::timeToMinBar(uDate, uOnlyMin);	// 时间到了第二天

	uint64_t lastTime = klineData->time(-1);						// 最后一根K线的时间
	uint32_t lastDate = klineData->date(-1);						// 最后一根K线的日期
	if (lastTime == INVALID_UINT32 || uBarTime > lastTime || tick->tradingdate() > lastDate)
	{
		//如果时间不一致,则新增一条K线
		WTSBarStruct *day = new WTSBarStruct;		// 用 tick 数据更新 bar 并返回
		day->date = tick->tradingdate();
		day->time = uBarTime;
		day->open = tick->price();
		day->high = tick->price();
		day->low = tick->price();
		day->close = tick->price();
		day->vol = tick->volume();
		day->money = tick->turnover();
		day->hold = tick->openinterest();
		day->add = tick->additional();

		return day;
	}
	else if (lastTime != INVALID_UINT32 && uBarTime < lastTime)
	{
		//这种情况主要为了防止日期反复出现
		return NULL;
	}
	else
	{
		// 用 tick 数据更新k线最后一个bar
		WTSBarStruct *bar = klineData->at(klineData->size()-1);
		bar->close = tick->price();
		bar->high = max(bar->high,tick->price());
		bar->low = min(bar->low,tick->price());
		bar->vol += tick->volume();
		bar->money += tick->turnover();
		bar->hold = tick->openinterest();
		bar->add += tick->additional();

		return NULL;
	}
}

// bar 数据更新5分钟的K线
WTSBarStruct* WTSDataFactory::updateMin5Data(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSBarStruct* newBasicBar)
{
	if (sInfo == NULL)
		return NULL;

	if (klineData->times() == 1)
	{
		klineData->appendBar(*newBasicBar);
		return klineData->at(-1);
	}

	//计算时间步长
	uint32_t steplen = 5 * klineData->times();

	const WTSBarStruct& curBar = *newBasicBar;

	uint32_t uTradingDate = curBar.date;						// bar 的日期
	uint32_t uDate = TimeUtils::minBarToDate(curBar.time);		// bar 的时间
	if (uDate == 19900000)
		uDate = uTradingDate;
	uint32_t uTime = TimeUtils::minBarToTime(curBar.time);
	uint32_t uMinute = sInfo->timeToMinutes(uTime) - 5;			// 将时间变为分钟再 -5


	uint32_t uBarMin = (uMinute / steplen)*steplen + steplen;
	uint64_t uBarTime = sInfo->minuteToTime(uBarMin);
	uBarTime = TimeUtils::timeToMinBar(uDate, (uint32_t)uBarTime);

	WTSBarStruct* lastBar = NULL;
	if (klineData->size() > 0)
	{
		lastBar = klineData->at(klineData->size() - 1);
	}

	bool bNewBar = false;
	if (lastBar == NULL || lastBar->date != uDate || lastBar->time != uBarTime)
	{

		//只要日期和时间都不符,则认为已经是一条新的bar了
		lastBar = new WTSBarStruct();
		bNewBar = true;

		memcpy(lastBar, &curBar, sizeof(WTSBarStruct));
		lastBar->date = uTradingDate;
		lastBar->time = uBarTime;
	}
	else
	{
		bNewBar = false;

		lastBar->high = max(lastBar->high, curBar.high);
		lastBar->low = min(lastBar->low, curBar.low);
		lastBar->close = curBar.close;
		lastBar->settle = curBar.settle;

		lastBar->vol += curBar.vol;
		lastBar->money += curBar.money;
		lastBar->add += curBar.add;
		lastBar->hold = curBar.hold;
	}
	// 判断k线是否闭合
	if (lastBar->time > curBar.time)
	{
		klineData->setClosed(false);
	}
	else
	{
		klineData->setClosed(true);
	}
	// 判断是否生成新 bar
	if (bNewBar)
	{
		klineData->appendBar(*lastBar);
		delete lastBar;

		return klineData->at(-1);
	}

	return NULL;
}

// 用 tick 更新 5分钟K线
WTSBarStruct* WTSDataFactory::updateMin5Data(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSTickData* tick)
{
	uint32_t steplen = 5*klineData->times();

	uint32_t uDate = tick->actiondate();
	uint32_t uTime = tick->actiontime()/100000;
	uint32_t uMinute = sInfo->timeToMinutes(uTime);
	if (sInfo->isLastOfSection(uTime))
	{
		uMinute--;
	}
	uint32_t uBarMin = (uMinute/steplen)*steplen + steplen;
	uint32_t uOnlyMin = sInfo->minuteToTime(uBarMin);
	if (uOnlyMin == 0)
	{
		uDate = TimeUtils::getNextDate(uDate);
	}
	uint64_t uBarTime = TimeUtils::timeToMinBar(uDate, uOnlyMin);

	uint64_t lastTime = klineData->time(klineData->size()-1);
	if(lastTime == INVALID_UINT32 || uBarTime != lastTime)
	{
		//如果时间不一致,则新增一条K线
		WTSBarStruct *day = new WTSBarStruct;
		day->date = tick->tradingdate();
		day->time = uBarTime;
		day->open = tick->price();
		day->high = tick->price();
		day->low = tick->price();
		day->close = tick->price();
		day->vol = tick->volume();
		day->money = tick->turnover();
		day->hold = tick->openinterest();
		day->add = tick->additional();

		return day;
	}
	else
	{
		WTSBarStruct *bar = klineData->at(klineData->size()-1);
		bar->close = tick->price();
		bar->high = max(bar->high,tick->price());
		bar->low = min(bar->low,tick->price());
		bar->vol += tick->volume();
		bar->money += tick->turnover();
		bar->hold = tick->openinterest();
		bar->add = tick->additional();

		return NULL;
	}
}

// 用 tick 更新日K
WTSBarStruct* WTSDataFactory::updateDayData(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSTickData* tick)
{
	uint32_t curDate = tick->tradingdate();
	uint32_t lastDate = klineData->date(klineData->size()-1);

	if(lastDate == INVALID_UINT32 || curDate != lastDate)
	{
		//如果时间不一致,则新增一条K线
		WTSBarStruct *day = new WTSBarStruct;
		day->date = curDate;
		day->time = 0;
		day->open = tick->price();
		day->high = tick->price();
		day->low = tick->price();
		day->close = tick->price();
		day->vol = tick->volume();
		day->money = tick->turnover();
		day->hold = tick->openinterest();
		day->add = tick->additional();

		return day;
	}
	else
	{
		WTSBarStruct *bar = klineData->at(klineData->size()-1);
		bar->close = tick->price();
		bar->high = max(bar->high,tick->price());
		bar->low = min(bar->low,tick->price());
		bar->vol += tick->volume();
		bar->money += tick->turnover();
		bar->hold = tick->openinterest();
		bar->add += tick->additional();

		return NULL;
	}
}

// 用 tick 更新秒K
WTSBarStruct* WTSDataFactory::updateSecData(WTSSessionInfo* sInfo, WTSKlineData* klineData, WTSTickData* tick)
{
	uint32_t seconds = klineData->times();
	uint32_t curSeconds = sInfo->timeToSeconds(tick->actiontime()/1000);
	uint32_t barSeconds = (curSeconds/seconds)*seconds + seconds;
	uint32_t barTime = sInfo->secondsToTime(barSeconds);

	if(klineData->isUnixTime())
	{
		uint32_t uDate = tick->actiondate();
		if (barTime < tick->actiontime() / 1000)
			uDate = TimeUtils::getNextDate(uDate);
		barTime = (uint32_t)(TimeUtils::makeTime(uDate, barTime * 1000) / 1000);
	}	

	uint64_t lastTime = klineData->time(klineData->size()-1);
	if(lastTime == INVALID_UINT32 || lastTime != barTime)
	{
		WTSBarStruct *day = new WTSBarStruct;
		day->date = tick->tradingdate();
		day->time = barTime;
		day->open = tick->price();
		day->high = tick->price();
		day->low = tick->price();
		day->close = tick->price();
		day->vol = tick->volume();
		day->money = tick->turnover();
		day->hold = tick->openinterest();
		day->add = tick->additional();

		return day;
	}
	else
	{
		WTSBarStruct *bar = klineData->at(klineData->size()-1);
		bar->close = tick->price();
		bar->high = max(bar->high,tick->price());
		bar->low = min(bar->low,tick->price());
		bar->vol += tick->volume();
		bar->money += tick->turnover();
		bar->hold = tick->openinterest();
		bar->add += tick->additional();

		return NULL;
	}
}

// 获取前一个周期的分钟
uint32_t WTSDataFactory::getPrevMinute(uint32_t curMinute, int period /* = 1 */)
{
	uint32_t h = curMinute/100;
	uint32_t m = curMinute%100;
	if(m == 0)
	{
		m = 60;
		if(h == 0) h = 24;

		return (h-1)*100 + (m-period);
	}
	else
	{
		return h*100 + m - period;
	}
}

WTSKlineData* WTSDataFactory::extractKlineData(WTSKlineSlice* baseKline, WTSKlinePeriod period, uint32_t times, WTSSessionInfo* sInfo, bool bIncludeOpen /* = true */)
{
	if(baseKline == NULL || baseKline->size() == 0)
		return NULL;

	//一倍,则不需要转换
	if(times <= 1 || period == KP_Tick)
	{
		return NULL;
	}

	if(period == KP_DAY)
	{
		return extractDayData(baseKline, times, bIncludeOpen);
	}
	else if(period == KP_Minute1)
	{
		return extractMin1Data(baseKline, times, sInfo, bIncludeOpen);
	}
	else if(period == KP_Minute5)
	{
		return extractMin5Data(baseKline, times, sInfo, bIncludeOpen);
	}
	
	return NULL;
}

WTSKlineData* WTSDataFactory::extractMin1Data(WTSKlineSlice* baseKline, uint32_t times, WTSSessionInfo* sInfo, bool bIncludeOpen /* = true */)
{
	//根据合约代码获取市场信息
	if(sInfo == NULL)
		return NULL;

	//计算时间步长
	uint32_t steplen = times;

	WTSKlineData* ret = WTSKlineData::create(baseKline->code(), 0);
	ret->setPeriod(KP_Minute1, times);

	for (auto i = 0; i < baseKline->size(); i++)
	{
		const WTSBarStruct& curBar = *baseKline->at(i);

		uint32_t uTradingDate = curBar.date;
		uint32_t uDate = TimeUtils::minBarToDate(curBar.time);
		if(uDate == 19900000)
			uDate = uTradingDate;
		uint32_t uTime = TimeUtils::minBarToTime(curBar.time);
		uint32_t uMinute = sInfo->timeToMinutes(uTime)-1;

		uint32_t uBarMin = (uMinute/steplen)*steplen + steplen;
		uint64_t uBarTime = sInfo->minuteToTime(uBarMin);
		//if(uBarTime > uTime && !sInfo->isInAuctionTime(uTime))
		//{
		//	//这种情况只可能是日期倒退
		//	uDate = TimeUtils::getNextDate(uDate, -1);
		//}
		uBarTime = TimeUtils::timeToMinBar(uDate, (uint32_t)uBarTime);

		WTSBarStruct* lastBar = NULL;
		if(ret->size() > 0)
		{
			lastBar = ret->at(ret->size()-1);
		}

		bool bNewBar = false;
		if(lastBar == NULL || lastBar->date != uDate || lastBar->time != uBarTime)
		{
			//只要日期和时间都不符,则认为已经是一条新的bar了
			lastBar = new WTSBarStruct();
			bNewBar = true;

			memcpy(lastBar, &curBar, sizeof(WTSBarStruct));
			lastBar->date = uDate;
			lastBar->time = uBarTime;
		}
		else
		{
			bNewBar = false;

			lastBar->high = max(lastBar->high, curBar.high);
			lastBar->low = min(lastBar->low, curBar.low);
			lastBar->close = curBar.close;
			lastBar->settle = curBar.settle;

			lastBar->vol += curBar.vol;
			lastBar->money += curBar.money;
			lastBar->add += curBar.add;
			lastBar->hold = curBar.hold;
		}

		if(bNewBar)
		{
			ret->appendBar(*lastBar);
			delete lastBar;
		}
	}

	//检查最后一条数据
	{
		WTSBarStruct* lastRawBar = baseKline->at(-1);
		WTSBarStruct* lastDesBar = ret->at(-1);
		//如果目标K线的最后一条数据的日期或者时间大于原始K线最后一条的日期或时间
		if ( lastDesBar->date > lastRawBar->date || lastDesBar->time > lastRawBar->time)
		{
			if (!bIncludeOpen)
				ret->getDataRef().resize(ret->size() - 1);
			else
				ret->setClosed(false);
		}
	}

	return ret;
}

WTSKlineData* WTSDataFactory::extractMin5Data(WTSKlineSlice* baseKline, uint32_t times, WTSSessionInfo* sInfo, bool bIncludeOpen /* = true */)
{
	if(sInfo == NULL)
		return NULL;

	//计算时间步长
	uint32_t steplen = 5*times;

	WTSKlineData* ret = WTSKlineData::create(baseKline->code(), 0);
	ret->setPeriod(KP_Minute5, times);

	for (auto i = 0; i < baseKline->size(); i++)
	{
		const WTSBarStruct& curBar = *baseKline->at(i);

		uint32_t uTradingDate = curBar.date;
		uint32_t uDate = TimeUtils::minBarToDate(curBar.time);
		if(uDate == 19900000)
			uDate = uTradingDate;
		uint32_t uTime = TimeUtils::minBarToTime(curBar.time);
		uint32_t uMinute = sInfo->timeToMinutes(uTime)-5;


		uint32_t uBarMin = (uMinute/steplen)*steplen+steplen;
		uint64_t uBarTime = sInfo->minuteToTime(uBarMin);
		uBarTime = TimeUtils::timeToMinBar(uDate, (uint32_t)uBarTime);

		WTSBarStruct* lastBar = NULL;
		if(ret->size() > 0)
		{
			lastBar = ret->at(ret->size()-1);
		}

		bool bNewBar = false;
		if(lastBar == NULL || lastBar->date != uDate || lastBar->time != uBarTime)
		{
			//只要日期和时间都不符,则认为已经是一条新的bar了
			lastBar = new WTSBarStruct();
			bNewBar = true;

			memcpy(lastBar, &curBar, sizeof(WTSBarStruct));
			lastBar->date = uTradingDate;
			lastBar->time = uBarTime;
		}
		else
		{
			bNewBar = false;

			lastBar->high = max(lastBar->high, curBar.high);
			lastBar->low = min(lastBar->low, curBar.low);
			lastBar->close = curBar.close;
			lastBar->settle = curBar.settle;

			lastBar->vol += curBar.vol;
			lastBar->money += curBar.money;
			lastBar->add += curBar.add;
			lastBar->hold = curBar.hold;
		}

		if(bNewBar)
		{
			ret->appendBar(*lastBar);
			delete lastBar;
		}
	}

	//检查最后一条数据
	{
		WTSBarStruct* lastRawBar = baseKline->at(-1);
		WTSBarStruct* lastDesBar = ret->at(-1);
		//如果目标K线的最后一条数据的日期或者时间大于原始K线最后一条的日期或时间
		if (lastDesBar->date > lastRawBar->date || lastDesBar->time > lastRawBar->time)
		{
			if (!bIncludeOpen)
				ret->getDataRef().resize(ret->size() - 1);
			else
				ret->setClosed(false);
		}
	}

	return ret;
}

WTSKlineData* WTSDataFactory::extractDayData(WTSKlineSlice* baseKline, uint32_t times, bool bIncludeOpen /* = true */)
{
	//计算时间步长
	uint32_t steplen = times;

	WTSKlineData* ret = WTSKlineData::create(baseKline->code(), 0);
	ret->setPeriod(KP_DAY, times);

	uint32_t count = 0;
	for (auto i = 0; i < baseKline->size(); i++, count++)
	{
		const WTSBarStruct& curBar = *baseKline->at(i);

		uint32_t uDate = curBar.date;

		WTSBarStruct* lastBar = NULL;
		if(ret->size() > 0)
		{
			lastBar = ret->at(ret->size()-1);
		}

		bool bNewBar = false;
		if(lastBar == NULL || count == steplen)
		{
			//只要日期和时间都不符,则认为已经是一条新的bar了
			lastBar = new WTSBarStruct();
			bNewBar = true;

			memcpy(lastBar, &curBar, sizeof(WTSBarStruct));
			lastBar->date = uDate;
			lastBar->time = 0;
			count = 0;
		}
		else
		{
			bNewBar = false;

			lastBar->high = max(lastBar->high, curBar.high);
			lastBar->low = min(lastBar->low, curBar.low);
			lastBar->close = curBar.close;
			lastBar->settle = curBar.settle;

			lastBar->vol += curBar.vol;
			lastBar->money += curBar.money;
			lastBar->add = curBar.add;
			lastBar->hold = curBar.hold;
		}

		if(bNewBar)
		{
			ret->appendBar(*lastBar);
			delete lastBar;
		}
	}

	return ret;
}

WTSKlineData* WTSDataFactory::extractKlineData(WTSTickSlice* ayTicks, uint32_t seconds, WTSSessionInfo* sInfo, bool bUnixTime /* = false */)
{
	if(ayTicks == NULL || ayTicks->size() == 0)
		return NULL;
	
	const WTSTickStruct& firstTick = *(ayTicks->at(0));

	if(sInfo == NULL)
		return NULL;

	WTSKlineData* ret = WTSKlineData::create(firstTick.code,0);
	ret->setPeriod(KP_Tick, seconds);
	ret->setUnixTime(bUnixTime);

	for (uint32_t i = 0; i < ayTicks->size(); i++)
	{
		WTSBarStruct* lastBar = NULL;
		if(ret->size() > 0)
		{
			lastBar = ret->at(ret->size()-1);
		}

		const WTSTickStruct* curTick = ayTicks->at(i);
		uint32_t uDate = curTick->trading_date;
		uint32_t curSeconds = sInfo->timeToSeconds(curTick->action_time/1000);
		uint32_t barSeconds = (curSeconds/seconds)*seconds + seconds;
		uint32_t barTime = sInfo->secondsToTime(barSeconds);

		//如果计算出来的K线时间戳小于tick数据的时间戳
		if(bUnixTime)
		{
			uint32_t actDt = curTick->action_date;
			if (barTime < curTick->action_time / 1000)
			{
				actDt = TimeUtils::getNextDate(actDt);
			}
			barTime = (uint32_t)(TimeUtils::makeTime(actDt, barTime * 1000) / 1000);
		}

		bool bNewBar = false;
		if (lastBar == NULL || uDate != lastBar->date || barTime != lastBar->time)
		{
			lastBar = new WTSBarStruct();
			bNewBar = true;

			lastBar->date = uDate;
			lastBar->time = barTime;

			lastBar->open = curTick->price;
			lastBar->high = curTick->price;
			lastBar->low = curTick->price;
			lastBar->close = curTick->price;
			lastBar->vol = curTick->volume;
			lastBar->money = curTick->turn_over;
			lastBar->hold = curTick->open_interest;
			lastBar->add = curTick->diff_interest;
		}
		else
		{
			lastBar->close = curTick->price;
			lastBar->high = max(lastBar->high,curTick->price);
			lastBar->low = min(lastBar->low,curTick->price);
			lastBar->vol += curTick->volume;
			lastBar->money += curTick->turn_over;
			lastBar->hold = curTick->open_interest;
			lastBar->add += curTick->diff_interest;
		}

		if(bNewBar)
		{
			ret->appendBar(*lastBar);
			delete lastBar;
		}
	}

	return ret;
}

bool WTSDataFactory::mergeKlineData(WTSKlineData* klineData, WTSKlineData* newKline)
{
	if (klineData == NULL || newKline == NULL)
		return false;

	if (strcmp(klineData->code(), newKline->code()) != 0)
		return false;

	if (!(klineData->period() == newKline->period() && klineData->times() == newKline->times()))
		return false;

	WTSKlineData::WTSBarList& bars = klineData->getDataRef();
	WTSKlineData::WTSBarList& newBars = newKline->getDataRef();
	if(bars.empty())
	{
		bars.swap(newBars);
		newBars.clear();
		return true;
	}
	else
	{
		uint64_t sTime,eTime;
		if(klineData->period() == KP_DAY)
		{
			sTime = bars[0].date;
			eTime = bars[bars.size() - 1].date;
		}
		else
		{
			sTime = bars[0].time;
			eTime = bars[bars.size() - 1].time;
		}

		WTSKlineData::WTSBarList tempHead, tempTail;
		uint32_t count = newKline->size();
		for (uint32_t i = 0; i < count; i++)
		{
			WTSBarStruct& curBar = newBars[i];

			uint64_t curTime;
			if (klineData->period() == KP_DAY)
				curTime = curBar.date;
			else
				curTime = curBar.time;

			if(curTime < sTime)
			{
				tempHead.emplace_back(curBar);
			}
			else if(curTime > eTime)
			{
				tempTail.emplace_back(curBar);
			}
		}

		bars.insert(bars.begin(), tempHead.begin(), tempHead.end());
		bars.insert(bars.end(), tempTail.begin(), tempTail.end());
	}
	
	return true;
}

```