
#### 服务原理以及C编写服务 ####



```

#include <Windows.h>
#include <iostream>
#include <string>

#pragma comment(lib, "Advapi32")


//全局变量
SERVICE_STATUS ServiceStatus; 
SERVICE_STATUS_HANDLE hStatus; 
std::string serviceName = "MyService9";
std::string displayName = "BBBB";

//服务程序函数
void ServiceMain(int argc, char** argv); 
void ControlHandler(DWORD request); 

//服务控制程序函数
void InstallService();
void  UninstallService();
void StopService();
void StartService2();

//自定义函数
int Init();



int main(int argc, char* argv[])
{
	SERVICE_TABLE_ENTRY ServiceTable[2];
	ServiceTable[0].lpServiceName = "";
	ServiceTable[0].lpServiceProc = (LPSERVICE_MAIN_FUNCTION)ServiceMain;

	ServiceTable[1].lpServiceName = NULL;
	ServiceTable[1].lpServiceProc = NULL;
	// 启动服务的控制分派机线程
	StartServiceCtrlDispatcher(ServiceTable); 


	std::cout<<"install the service input—————>i"<<std::endl;
	std::cout<<"uninstall the service input————>u"<<std::endl;
	std::cout<<"start the service input——————>start"<<std::endl;
	std::cout<<"stop the service input——————->stop"<<std::endl;
	std::cout<<"quit input ————————————->q"<<std::endl;
	std::cout<<"——>";
	std::string type;
	while(std::cin>>type)
	{
		if(type == "i")
			InstallService();
		else if(type == "u")
		{
			StopService();
			UninstallService();
		}
		else if(type == "stop")
			StopService();
		else if(type == "start") 
			StartService2();
		else if(type == "q")
			break;
		
		std::cout<<"——>";
	}

	return 0;
}


/*******************************************服务程序*******************************************/


void ServiceMain(int argc, char** argv) 
{ 
	int error; 

	ServiceStatus.dwServiceType = SERVICE_WIN32; 
	ServiceStatus.dwCurrentState = SERVICE_START_PENDING; 
	ServiceStatus.dwControlsAccepted   = SERVICE_ACCEPT_STOP | SERVICE_ACCEPT_SHUTDOWN;
	ServiceStatus.dwWin32ExitCode = 0; 
	ServiceStatus.dwServiceSpecificExitCode = 0; 
	ServiceStatus.dwCheckPoint = 0; 
	ServiceStatus.dwWaitHint = 0; 

	hStatus = RegisterServiceCtrlHandler("", (LPHANDLER_FUNCTION)ControlHandler); 
	if (hStatus == (SERVICE_STATUS_HANDLE)0) 
	{
		// Registering Control Handler failed
		return; 
	} 

	// Initialize Service 
	error = Init(); 
	if (!error) 
	{
		// Initialization failed
		ServiceStatus.dwCurrentState = SERVICE_STOPPED; 
		ServiceStatus.dwWin32ExitCode = -1; 
		SetServiceStatus(hStatus, &ServiceStatus); 
		return; 
	} 

	// We report the running status to SCM. 
	ServiceStatus.dwCurrentState = SERVICE_RUNNING; 
	SetServiceStatus (hStatus, &ServiceStatus);

	//now, do what you want
	......

	return; 
}

void ControlHandler(DWORD request) 
{ 
	switch(request) 
	{ 
	case SERVICE_CONTROL_STOP: 
		ServiceStatus.dwWin32ExitCode = 0; 
		ServiceStatus.dwCurrentState = SERVICE_STOPPED; 
		SetServiceStatus (hStatus, &ServiceStatus);

		//做一些善后工作
		......

		return; 

	case SERVICE_CONTROL_SHUTDOWN: 
		ServiceStatus.dwWin32ExitCode = 0; 
		ServiceStatus.dwCurrentState = SERVICE_STOPPED; 
		SetServiceStatus (hStatus, &ServiceStatus);
		return; 

	default:
		break;
	} 

	// Report current status
	SetServiceStatus (hStatus, &ServiceStatus);
	return; 
}


/*******************************************服务控制程序*******************************************/


//安装服务
void InstallService()
{
	SC_HANDLE schSCManager;
	SC_HANDLE schService;
	char ch[300] = {0};

	//第一参数NULL，GetModuleFileName取出当前进程的exe文件全路径，如E:\projects\Win32Service2\Release\Win32Service2.exe
	if(!GetModuleFileName( NULL, ch, 300))
	{
		printf("Cannot install service (%d)\n", GetLastError());
		return;
	}

	//获取指定SCM（服务控制管理器）的句柄,参数1指定计算机名称、为NULL则取本机SCM句柄，参数3为权限、SC_MANAGER_ALL_ACCESS即可
	schSCManager = OpenSCManager( 
		NULL,                    // local computer
		NULL,                    // ServicesActive database 
		SC_MANAGER_ALL_ACCESS);  // full access rights 

	if (NULL == schSCManager) 
	{
		printf("OpenSCManager failed (%d)\n", GetLastError());
		return;
	}

	// Create the service
	schService = CreateService( 
		schSCManager,              // SCM database 
		serviceName.c_str(),                   // name of service 
		displayName.c_str(),                   // service name to display 
		SERVICE_ALL_ACCESS,        // desired access 
		SERVICE_WIN32_OWN_PROCESS, // service type 
		SERVICE_AUTO_START,      // start type 
		SERVICE_ERROR_NORMAL,      // error control type 
		"E:\\projects\\attemp3\\Debug\\attemp3.exe",                    // path to service's binary 
		NULL,                      // no load ordering group 
		NULL,                      // no tag identifier 
		NULL,                      // no dependencies 
		NULL,                      // LocalSystem account 
		NULL);                     // no password 

	if (schService == NULL) 
	{
		printf("CreateService failed (%d)\n", GetLastError()); 
		CloseServiceHandle(schSCManager);
		return;
	}
	else printf("Service installed successfully\n"); 

	CloseServiceHandle(schService); 
	CloseServiceHandle(schSCManager);
}


//卸载服务
void UninstallService()
{
	SC_HANDLE schSCManager = OpenSCManager(NULL, NULL, SC_MANAGER_ALL_ACCESS);
	if(schSCManager == NULL)
	{
		printf("OpenSCManager failed (%d)\n", GetLastError()); 
		return;
	}

	SC_HANDLE schService = OpenService(schSCManager, serviceName.c_str(), SC_MANAGER_ALL_ACCESS);
	if(schService == NULL)
	{
		printf("OpenService failed (%d)\n", GetLastError());
		CloseServiceHandle(schSCManager);
		return;
	}

	//卸载服务
	if(DeleteService(schService))
		std::cout<<"uninstall service ok"<<std::endl;
	else
		std::cout<<"uninstall service fail"<<std::endl;

	CloseServiceHandle(schService);
	CloseServiceHandle(schSCManager);
}


//开启服务
void StartService2()
{
	//打开SCM，取得SCM句柄
	SC_HANDLE schSCManager = OpenSCManager(NULL, NULL, SC_MANAGER_ALL_ACCESS);
	if(schSCManager == NULL)
	{
		printf("OpenSCManager failed (%d)\n", GetLastError()); 
		return;
	}

	//打开服务，取得服务句柄。serviceName为要打开的服务名称，对应着CreateService创建服务时起的名字。
	SC_HANDLE schService = OpenService(schSCManager, serviceName.c_str(), SC_MANAGER_ALL_ACCESS);
	if(schService == NULL)
	{
		printf("OpenService failed (%d)\n", GetLastError());
		CloseServiceHandle(schSCManager);
		return;
	}

	//开启服务
	SERVICE_STATUS serviceStatus;
	if(StartService(schService, NULL, NULL))
	{
		//查询服务当前的状态，等待启动服务
		while(QueryServiceStatus(schService, &serviceStatus))
		{
			if(serviceStatus.dwCurrentState == SERVICE_RUNNING)
				break;
			else
				Sleep(100);
		}
		std::cout<<"Service Start Successfully"<<std::endl;
	}

	//妥善关闭所有打开的句柄，防止待会卸载服务时卸载不掉的麻烦
	CloseServiceHandle(schService);
	CloseServiceHandle(schSCManager);
}


//停止服务
void StopService()
{
	SC_HANDLE schSCManager = OpenSCManager(NULL, NULL, SC_MANAGER_ALL_ACCESS);
	if(schSCManager == NULL)
	{
		printf("OpenSCManager failed (%d)\n", GetLastError()); 
		return;
	}

	SC_HANDLE schService = OpenService(schSCManager, serviceName.c_str(), SC_MANAGER_ALL_ACCESS);
	if(schService == NULL)
	{
		printf("OpenService failed (%d)\n", GetLastError());
		CloseServiceHandle(schSCManager);
		return;
	}

	//关闭服务
	SERVICE_STATUS serviceStatus;
	if(ControlService(schService, SERVICE_CONTROL_STOP, &serviceStatus))
	{
		//查询服务当前的状态，等待停止服务
		while(QueryServiceStatus(schService, &serviceStatus))
		{
			if(serviceStatus.dwCurrentState == SERVICE_STOPPED)
				break;
			else
				Sleep(100);
		}
		std::cout<<"Service Stop Successfully"<<std::endl;
	}

	CloseServiceHandle(schService);
	CloseServiceHandle(schSCManager);
}


/*******************************************自定义函数*******************************************/

//用来给109行自己程序运行前做些初始化工作，如果初始化很简单就没必要定义此函数
int Init(){
	return true;
}
```

#### 命令控制服务 ####

除了通过“控制面板”>“管理工具”>“服务”来查看服务之外，还有很多种其他的方式可以对Windows服务进行管理。在命令行方式下，你可以使用sc.exe（ServiceControl的缩写）来管理服务。

我们可以用sc.exe命令来查询、启动、停止，甚至删除服务。

点击 开始 > 运行 > 输入"cmd"回车，然后在弹出的DOS窗口中输入sc回车就可以看到sc命令的使用帮助了。

sc命令的语法格式：

sc [command] [servicename] ...

sc命令使用例子：

sc query
查看所有服务的运行状态

sc query 服务名
查看某个服务的运行状态。

sc qc 服务名
查看某个服务的配置信息。

sc start 服务名
启动服务。例如启动apache2.2服务器，就写成scstartapache2.2。

sc stop 服务名
停止服务。例如sc stop apache2.2。

sc delete 服务名
删除服务。例如sc delete apache2.2。

sc config 服务名 start=auto|demand|disabled
修改服务启动类型。start参数的值可以是demand（手动）、disabled（禁用），auto（自动）。
例如sc config apache2.2 start=demand，将apache设置为手动启动。
特别注意：start=后面有一个空格

使用提示

如果服务名称中包含有空格，记得在服务名称上加引号。例如sc stop "myservice"。
“服务名称”和“服务显示名称”是不一样的。sc指令使用的是“服务名称”。
我们通过控制面板=>“管理工具"=>打开"服务"，我们看到服务的显示名称，双击打开某个服务可以看到真正的服务名字。
sc start和sc stop功能上类似于net start和net stop，但速度更快且能停止的服务更多。
sc delete命令的实质都是删除HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services下的ServiceName分支。所以你也可以用reg命令删除名为ServiceName的服务：
regdeleteHKLM\SYSTEM\CurrentControlSet\Services\ServiceName


参考文档:

http://www.cnblogs.com/hbccdf/p/3477644.html
https://blog.csdn.net/zhangpeng_linux/article/details/7001084
https://blog.csdn.net/rongxiaojun1989/article/details/30028239