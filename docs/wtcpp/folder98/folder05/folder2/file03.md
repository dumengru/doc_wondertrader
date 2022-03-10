# CTA调度器模板

source: `{{ page.path }}`

## ExpCtaMocker.h

内容非常简单, 继承父类CtaMocker, 接口中先执行父类相关函数, 然后执行执行引擎(WtBtRunner)相关函数

### ExpCtaMocker

```cpp
class ExpCtaMocker : public CtaMocker
{
public:
	ExpCtaMocker(HisDataReplayer* replayer, const char* name, int32_t slippage = 0, bool persistData = true, EventNotifier* notifier = NULL);
	virtual ~ExpCtaMocker();

public:
	virtual void on_init() override;
	virtual void on_session_begin(uint32_t uCurDate) override;
	virtual void on_session_end(uint32_t uCurDate) override;
	virtual void on_tick_updated(const char* stdCode, WTSTickData* newTick) override;
	virtual void on_bar_close(const char* stdCode, const char* period, WTSBarStruct* newBar) override;
	virtual void on_calculate(uint32_t curDate, uint32_t curTime) override;
	virtual void on_calculate_done(uint32_t curDate, uint32_t curTime) override;
	virtual void on_bactest_end() override;
};
```

### ExpCtaMocker.cpp

```cpp

extern WtBtRunner& getRunner();

ExpCtaMocker::ExpCtaMocker(HisDataReplayer* replayer, const char* name, int32_t slippage /* = 0 */, bool persistData/* = true*/, EventNotifier* notifier /* = NULL */)
	: CtaMocker(replayer, name, slippage, persistData, notifier)
{}

ExpCtaMocker::~ExpCtaMocker()
{}

// 1. 执行父类 on_init() 2. 执行引擎 ctx_on_init 回调 3. 执行引擎事件回调
void ExpCtaMocker::on_init()
{
	CtaMocker::on_init();
	// 向外部回调
	getRunner().ctx_on_init(_context_id, ET_CTA);
	getRunner().on_initialize_event();
}

void ExpCtaMocker::on_session_begin(uint32_t uCurDate)
{
	CtaMocker::on_session_begin(uCurDate);
	getRunner().ctx_on_session_event(_context_id, uCurDate, true, ET_CTA);
	getRunner().on_session_event(uCurDate, true);
}

void ExpCtaMocker::on_session_end(uint32_t uCurDate)
{
	CtaMocker::on_session_end(uCurDate);
	getRunner().ctx_on_session_event(_context_id, uCurDate, false, ET_CTA);
	getRunner().on_session_event(uCurDate, false);
}

void ExpCtaMocker::on_tick_updated(const char* stdCode, WTSTickData* newTick)
{
	CtaMocker::on_tick_updated(stdCode, newTick);
	getRunner().ctx_on_tick(_context_id, stdCode, newTick, ET_CTA);
}

void ExpCtaMocker::on_bar_close(const char* code, const char* period, WTSBarStruct* newBar)
{
	CtaMocker::on_bar_close(code, period, newBar);
	// 要向外部回调
	getRunner().ctx_on_bar(_context_id, code, period, newBar, ET_CTA);
}

void ExpCtaMocker::on_calculate(uint32_t curDate, uint32_t curTime)
{
	CtaMocker::on_calculate(curDate, curTime);
	getRunner().ctx_on_calc(_context_id, curDate, curTime, ET_CTA);
}

void ExpCtaMocker::on_calculate_done(uint32_t curDate, uint32_t curTime)
{
	CtaMocker::on_calculate_done(curDate, curTime);
	getRunner().ctx_on_calc_done(_context_id, curDate, curTime, ET_CTA);
	getRunner().on_schedule_event(curDate, curTime);
}

void ExpCtaMocker::on_bactest_end()
{
	getRunner().on_backtest_end();
}
```