# 定时器

`timer.h`

```cpp
#pragma once

#include "yunweibase.h"

class CTimer
{
public:
	CTimer();
	~CTimer();
private:
	timer_t timer;
	// 定时器状态
	bool running;
private:
	// 定时扫描间隔时间
	int interTime_sec;
	int interTime_nsec;
	// 定时扫描倒计时时间
	int valueTime_sec;
	int valueTime_nsec;
	struct sigevent evp;
	struct itimerspec ts;
private:
	void SetSigevent();
	void SetTime();
	// 启动/停止或重置定时器
	int Set();
	int Create();
	bool Destroy();
	void SetRunState(bool state);
public:
	// 先不考虑配置定时器时间
	void ConfigTime(int intTime_sec, int intTime_nsec, int valTime_sec, int valTime_nsec);
	void StartTimer();
	void StopTimer();
	// 获取定时器状态
	bool IsRunning();

	// 打印本地时间，含换行符
	static void PrintlnLocalTime();
	// 打印UTC时间，含换行符
	static void PrintlnUtcTime();
};

```

​                    



`timer.cpp`

```cpp
#include "timer.h"

#ifdef _WIN32
#pragma warning(disable:4996)
#endif

CTimer::CTimer()
{
	interTime_sec = 2;
	interTime_nsec = 0;
	valueTime_sec = 2;
	valueTime_nsec = 0;
	timer = 0;
	running = false;
	memset(&ts, 0, sizeof(ts));
	memset(&evp, 0, sizeof(evp));
}

CTimer::~CTimer()
{

}

void CTimer::SetSigevent()
{
	evp.sigev_value.sival_ptr = &timer;
	// 若为SIGEV_THREAD，则以sigev_notification_attributes为线程属性创建一个线程，线程的入口地址为sigev_notify_function，传入sigev_value作为一个参数。
	//evp.sigev_notify = SIGEV_THREAD;
	evp.sigev_notify = SIGEV_SIGNAL;
	evp.sigev_signo = SIGUSR1;
	signal(evp.sigev_signo, SignHandler);
}

void CTimer::SetTime()
{
	ts.it_interval.tv_sec = interTime_sec;
	ts.it_interval.tv_nsec = interTime_nsec;
	ts.it_value.tv_sec = valueTime_sec;
	ts.it_value.tv_nsec = valueTime_nsec;
}

void CTimer::ConfigTime(int intTime_sec, int intTime_nsec, int valTime_sec, int valTime_nsec)
{
	interTime_sec = intTime_sec;
	interTime_nsec = intTime_nsec;
	valueTime_sec = valTime_sec;
	valueTime_nsec = valTime_nsec;
}

int CTimer::Set()
{
	int ret;
	SetTime();
	// ret = timer_settime(timer, CLOCK_REALTIME, &ts, NULL);
	ret = timer_settime(timer, 0, &ts, NULL);
	if (ret) {
		perror("timer_settime");
	}
	return ret;
}

/*
	timer_create() creates a new per-process interval timer.
	Per-process timers are not inherited by a child process across a fork() and are disarmed and deleted by an exec.
	(定时器是每个进程自己的，不是在fork时继承的)
	[参考]:
	(1) https://man7.org/linux/man-pages/man2/timer_create.2.html
	(2) https://pubs.opengroup.org/onlinepubs/007908799/xsh/timer_create.html
*/
int CTimer::Create()
{
	int ret;
	SetSigevent();

	// 后面应考虑采用更稳定的时钟源 CLOCK_MONOTONIC
	//ret = timer_create(CLOCK_MONOTONIC, &evp, &timer);
	ret = timer_create(CLOCK_REALTIME, &evp, &timer);
	if (ret < 0)
	{
		perror("timer_create");
	}
	return ret;
}

bool CTimer::Destroy()
{
	// 定时器使用结束后,需要进行销毁
	int ret;
	ret = timer_delete(timer);
	if (ret < 0)
	{
		perror("timer_delete");
	}
	return true;
}

void CTimer::StartTimer()
{
	Create();
	Set();
	SetRunState(true);
}

void CTimer::StopTimer()
{
	Destroy();
	SetRunState(false);
}

bool CTimer::IsRunning()
{
	return running;
}

void CTimer::SetRunState(bool state)
{
	running = state;
}

void CTimer::PrintlnLocalTime()
{
	time_t result = time(NULL);
	if (result != (time_t)(-1))
	{
		// asctime: Return a string of the form "Day Mon dd hh:mm:ss yyyy\n" that is the representation of TP in this format.
		std::cout << asctime(localtime(&result));
	}
}

void CTimer::PrintlnUtcTime()
{
	time_t result = time(NULL);
	if (result != (time_t)(-1))
	{
		// asctime: Return a string of the form "Day Mon dd hh:mm:ss yyyy\n" that is the representation of TP in this format.
		std::cout << asctime(gmtime(&result));
	}
}

```

