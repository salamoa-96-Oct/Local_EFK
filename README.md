# Local_EFK

# 구성 환경(Version)

![Untitled](https://user-images.githubusercontent.com/83336692/147642899-67d26079-fb89-41a1-934c-b7f07d522bfc.png)


- 구동환경
    
    → virtual_box(Oracle)
    
    Ubuntu : 20.04 TLS 버전 사용
    
    Nat : 10.100.0.0/24  DHCP 사용
    
    → Kubernetes Version
    
    ```yaml
    $ kubectl cluster-info
    	Kubernetes control plane is running at https://10.100.0.104:6443
    	CoreDNS is running at https://10.100.0.104:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    	
    	To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
    
    $ kubectl version
    	Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.2", GitCommit:"8b5a19147530eaac9476b0ab82980b4088bbc1b2", GitTreeState:"clean", BuildDate:"2021-09-15T21:38:50Z", GoVersion:"go1.16.8", Compiler:"gc", Platform:"linux/amd64"}
    	Server Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.2", GitCommit:"8b5a19147530eaac9476b0ab82980b4088bbc1b2", GitTreeState:"clean", BuildDate:"2021-09-15T21:32:41Z", GoVersion:"go1.16.8", Compiler:"gc", Platform:"linux/amd64"}
    
    $ kubeadm version
    	kubeadm version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.2", GitCommit:"8b5a19147530eaac9476b0ab82980b4088bbc1b2", GitTreeState:"clean", BuildDate:"2021-09-15T21:37:34Z", GoVersion:"go1.16.8", Compiler:"gc", Platform:"linux/amd64"}
    ```
    

→ helm version : 3.7.1 version 사용

→ elasticsearch version : 7.15

→ fluentd version : 3.3

→ kibana version : 7.15

---
