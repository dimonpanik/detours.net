# detours.net
Détours.net use CLR as hooking engine. It's based on detours project from Microsoft and ability of CLR to generate transition stub for managed function to be called from unmanaged code.

*detours.net* is as simple to use as *DllImport* attribute for _pinvoke_ unmanaged code from managed code.

## Generate a plugin

Imagine you want to log all GUID of COM object use by a target application, like malware, you have to use *detours.net*.
First step is to generate a plugin. Plugin is simple a .net DLL linked with *detoursnet.dll* assembly.

Then you have to tell *detours.net* how and from which API you want to hook. You just have to declare a delegate which match your target function signature, and declare your associate hook like this :

```c#
// Declare your delegate
public delegate int CoCreateInstanceDelegate(
	Guid rclsid, IntPtr pUnkOuter, 
	int dwClsContext, Guid riid, ref IntPtr ppv
);

// And now declare your hook
[Detours("ole32.dll", typeof(CoCreateInstanceDelegate))]
public static int CoCreateInstance(
	Guid rclsid, IntPtr pUnkOuter,
	int dwClsContext, Guid riid, ref IntPtr ppv
)
{
	// Call real function
	int result = ((CoCreateInstanceDelegate)DelegateStore.GetReal(MethodInfo.GetCurrentMethod()))(rclsid, pUnkOuter, dwClsContext, riid, ref ppv);

	Console.WriteLine(" {" + rclsid.ToString() + "} {" + riid.ToString() + "} " + result.ToString("x"));
	
	return result;
}
```

That's all. Build your assembly *myplugin.dll*, and run it with *detoursnetruntime.exe*.

```bat
.\detoursNetRuntime myplugin.dll c:\windows\notepad.exe
```

## How does it works ?

### DetoursNetRuntime

*detours.net* is based on detours project from Microsoft, which is mostly use in API hooking. It create a process in suspended mode, and then rewrite the IAT to insert a new dll at first place. This implies that *Dllmain* of this dll will be execute first before all other code in your application. That's was be done by *detoursNetRuntime.exe*, but inject a special DLL called *detoursNetCLR.dll* described in next chapter.

### DetoursNetCLR

DetoursNetCLR.dll is in charge to load CLR and the DetoursNet.dll assembly in current process. To do that we use CLR hosting from COM Component. But this forbidden from *DllMain* because of *loader lock*. To work around this issue, we use *Detours* from microsoft to hook entry point of target process, and load CLR into new *main* function.

### DetoursNet
