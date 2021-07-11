# 客户端

**`client.h`**

```c++
#pragma once

#include "yunweibase.h"

class CClient
{
public:
	CClient();
	~CClient();
private:
	short family;
	char addr[32];
	u16 port;
	// 0-TCP,1-UDP
	int commType;
	u16 keepAliveSec;
	u16 reportCycleSec;
	int domain;
	int sockType;
	int protocol;

	int client_sockfd;
	struct sockaddr_in remote_addr; 

	struct timeval rcvTimeOut;
	u32 rcvTimeOutLen;
	struct timeval sndTimeOut;
	u32 sndTimeOutLen;
	struct timeval connectTimeOut;

	int rcvBuf;
	u32 rcvBufLen;
	int sndBuf;
	u32 sndBufLen;

private:
	bool socketState;
	bool connectState;
private:
	enum SettableSocketOption
	{
		kSetSoRcvTimeO,
		kSetSoSndTimeO,
	};
	enum GettableSocketOption
	{
		kGetSoRcvTimeO,
		kGetSoSndTimeO,
		kGetSoRcvBuf,
		kGetSoSndBuf,
	};
	void SetTimeOutValue(SettableSocketOption option, struct timeval timeOutValue);
	void SetOption(SettableSocketOption option);
	void GetOption(GettableSocketOption option);
	void SetClient();
	bool CreateSock();
	bool CloseSock();
	bool Connect();
	void SetSockState(bool state);
	void SetConnState(bool state);
	bool GetSockState();
	// 非阻塞connect
	int ConnectNonb(int sockfd, const sockaddr* saptr, socklen_t salen);
public:
	bool Send(char* buf, u32 len);
	bool Receive(char* buf, u32* len);
	bool StartLink();
	void StopLink();
	bool GetConnState();
	void Config(u16 keepAliveSec_, u16 reportCycleSec_, const char* addr_, unsigned short port_, int commType_);
};

```

​          

**`client.cpp`**

```c++
#include "client.h"

#ifdef _WIN32
#pragma warning(disable:4996)
#endif

CClient::CClient()
{
	family = AF_INET;
	memset(&remote_addr, 0, sizeof(remote_addr));
	memset(addr, 0, sizeof(addr));
	port = 0;
	keepAliveSec = 30;
	reportCycleSec = 300;

	domain = PF_INET;
	sockType = SOCK_STREAM;
	protocol = 0;
	commType = 0;

	socketState = false;
	connectState = false;

	// 默认设置接收阻塞时的等待时间 0.3s
	rcvTimeOut.tv_sec = 0;
	rcvTimeOut.tv_usec = 3e5;
	rcvTimeOutLen = sizeof(rcvTimeOut);
	// 默认设置发送阻塞时的等待时间 0.3s
	sndTimeOut.tv_sec = 0;
	sndTimeOut.tv_usec = 3e5;
	sndTimeOutLen = sizeof(sndTimeOut);
	// 默认设置连接阻塞时的等待时间 0.3s
	connectTimeOut.tv_sec = 0;
	connectTimeOut.tv_usec = 3e5;

	rcvBuf = 0;
	rcvBufLen = sizeof(rcvBuf);
	sndBuf = 0;
	sndBufLen = sizeof(rcvBuf);
}

CClient::~CClient()
{

}

void CClient::Config(u16 keepAliveSec_, u16 reportCycleSec_, const char* addr_, unsigned short port_, int commType_)
{
	keepAliveSec = keepAliveSec_;
	reportCycleSec = reportCycleSec_;
	strncpy(addr, addr_, sizeof(addr) - 1);
	port = port_;
	commType = commType_;
}

void CClient::SetClient()
{
	remote_addr.sin_family = family;
	// 服务器IP地址
	inet_pton(remote_addr.sin_family, addr, &remote_addr.sin_addr);
	// 服务器端口号
	remote_addr.sin_port = htons(port);
	// 通信类型
	if (commType == 0)
	{
		sockType = SOCK_STREAM;
	}
	else
	{
		sockType = SOCK_DGRAM;
	}
}

bool CClient::CreateSock()
{
	// 暂时先注释掉这里，以保证每次重连时可以创建出绝对可用的全新的套接字，虽然存在多次重连时可能会造成系统中生成了大量空闲的没有被关闭的套接字的隐患
	/*if (isSocket)
		return true;
	else*/
	{
		client_sockfd = socket(domain, sockType, protocol);
		if (client_sockfd < 0)
		{
			SetSockState(false);
			perror("YunWei: socket error");
			return false;
		}
		else
		{
			SetSockState(true);
			return true;
		}
	}
}

void CClient::SetTimeOutValue(SettableSocketOption option, struct timeval timeOutValue)
{
	switch (option)
	{
	case kSetSoRcvTimeO:
	{
		rcvTimeOut = timeOutValue;
		break;
	}
	case kSetSoSndTimeO:
	{
		sndTimeOut = timeOutValue;
		break;
	}
	}
}

void CClient::SetOption(SettableSocketOption option)
{
	switch (option)
	{
	case kSetSoRcvTimeO:
	{
		setsockopt(client_sockfd, SOL_SOCKET, SO_RCVTIMEO, (const void*)&rcvTimeOut, rcvTimeOutLen);
		break;
	}
	case kSetSoSndTimeO:
	{
		setsockopt(client_sockfd, SOL_SOCKET, SO_SNDTIMEO, (const void*)&sndTimeOut, sndTimeOutLen);
		break;
	}
	}
}

void CClient::GetOption(GettableSocketOption option)
{
	switch (option)
	{
	case kGetSoRcvTimeO:
	{
		getsockopt(client_sockfd, SOL_SOCKET, SO_RCVTIMEO, (void*)&rcvTimeOut, (socklen_t*)&rcvTimeOutLen);
		std::cout << "YunWei: " << "Receive timeout value = " << rcvTimeOut.tv_sec << "s" << rcvTimeOut.tv_usec << "us" << std::endl;
		break;
	}
	case kGetSoSndTimeO:
	{
		getsockopt(client_sockfd, SOL_SOCKET, SO_SNDTIMEO, (void*)&sndTimeOut, (socklen_t*)&sndTimeOutLen);
		std::cout << "YunWei: " << "Send timeout value = " << sndTimeOut.tv_sec << "s" << sndTimeOut.tv_usec << "us" << std::endl;
		break;
	}
	case kGetSoRcvBuf:
	{
		getsockopt(client_sockfd, SOL_SOCKET, SO_RCVBUF, (void*)&rcvBuf, (socklen_t*)&rcvBufLen);
		std::cout << "YunWei: " << "receive buffer size = " << rcvBuf << std::endl;
		break;
	}
	case kGetSoSndBuf:
	{
		getsockopt(client_sockfd, SOL_SOCKET, SO_SNDBUF, (void*)&sndBuf, (socklen_t*)&sndBufLen);
		std::cout << "YunWei: " << "send buffer size = " << sndBuf << std::endl;
		break;
	}
	}
}

bool CClient::CloseSock()
{
	int ret;
	// 关闭套接字
	ret = close(client_sockfd);
	if (ret < 0)
	{
		perror("close sockfd");
	}
	// 这里存在缺陷，当链路异常断开时，isSocket和connected会没有被置为false
	SetSockState(false);
	SetConnState(false);
	return true;
}

bool CClient::Connect()
{
	if (!GetSockState())
	{
		SetConnState(false);
		return false;
	}
	std::cout << "YunWei: " << "trying to connect server" << "(ip=" << addr << ":" << port << ")" << std::endl;
	// TCP
	if (commType == 0)
	{
		// 将套接字绑定到服务器的网络地址上
		if (ConnectNonb(client_sockfd, (struct sockaddr*)&remote_addr, sizeof(struct sockaddr)) < 0)
		{
			SetConnState(false);
			perror("YunWei: connect fail");
			return false;
		}
		else
		{
			SetConnState(true);
			std::cout << "YunWei: " << "connect to TCP server successfully" << std::endl;
			return true;
		}
	}
	// UDP
	else
	{
		SetConnState(true);
		std::cout << "YunWei: " << "connect to UDP server successfully" << std::endl;
		return true;
	}
}

bool CClient::Send(char* buf, u32 len)
{
	if (GetConnState())
	{
		// TCP
		if (commType == 0)
		{
			int length = -1;
			length = send(client_sockfd, buf, len, 0);
			if (length < 0)
			{
				perror("YunWei: send fail");
				return false;
			}
			else
			{
				return true;
			}
		}
		// UDP
		else
		{
			int length = -1;
			int sin_size = sizeof(struct sockaddr_in);
			length = sendto(client_sockfd, buf, len, 0, (struct sockaddr*)&remote_addr, sin_size);
			if (length < 0)
			{
				perror("YunWei: send fail");
				return false;
			}
			else
			{
				return true;
			}
		}
	}
	else
	{
		return false;
	}
}

bool CClient::Receive(char* buf, u32* len)
{
	if (GetConnState())
	{
		// TCP
		if (commType == 0)
		{
			int length = -1;
			length = recv(client_sockfd, buf, kBufSize, 0);
			if (length < 0)
			{
				//perror("YunWei: recv fail");
				return false;
			}
			else
			{
				*len = length;
				return true;
			}
		}
		// UDP
		else
		{
			int length = -1;
			u32 sin_size = sizeof(struct sockaddr_in);
			length = recvfrom(client_sockfd, buf, kBufSize, 0, (struct sockaddr*)&remote_addr, (socklen_t*)&sin_size);
			if (length < 0)
			{
				//perror("YunWei: recv fail");
				return false;
			}
			else
			{
				*len = length;
				return true;
			}
		}
	}
	else
	{
		return false;
	}
}

bool CClient::StartLink()
{
	bool result = false;
	SetClient();
	result = CreateSock();
	if (result)
	{
		SetOption(kSetSoRcvTimeO);
		SetOption(kSetSoSndTimeO);
		result = Connect();
	}
	return result;
}

void CClient::StopLink()
{
	CloseSock();
}

bool CClient::GetConnState()
{
	return connectState;
}

bool CClient::GetSockState()
{
	return socketState;
}

void CClient::SetConnState(bool state)
{
	connectState = state;
}

void CClient::SetSockState(bool state)
{
	socketState = state;
}

int CClient::ConnectNonb(int sockfd, const sockaddr* saptr, socklen_t salen)
{
	int flags, n, error;
	socklen_t len = 0;
	fd_set rset, wset;
	struct timeval tval = connectTimeOut;

	flags = fcntl(sockfd, F_GETFL, 0);
	fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);

	error = 0;
	if ((n = connect(sockfd, saptr, salen)) < 0)
		if (errno != EINPROGRESS)
			return -1;

	// connect completed immediately
	if (n == 0)
	{
		return 0;
	}

	FD_ZERO(&rset);
	FD_SET(sockfd, &rset);
	wset = rset;

	if ((n = select(sockfd + 1, &rset, &wset, NULL, &tval)) == 0)
	{
		// time out
		close(sockfd);
		errno = ETIMEDOUT;
		return -1;
	}

	if (FD_ISSET(sockfd, &rset) || FD_ISSET(sockfd, &wset))
	{
		len = sizeof(error);
		if (getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &error, &len) < 0)
			return -1;
	}
	else
	{
		std::cout << "YunWei: " << "select error: sockfd not set" << std::endl;
	}

	fcntl(sockfd, F_SETFL, flags);
	if (error)
	{
		close(sockfd);
		errno = error;
		return -1;
	}

	return 0;
}
```

