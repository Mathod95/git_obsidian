---
tags:
  - APP/ISTIO
source: https://adil.medium.com/how-to-mirror-traffic-to-another-pod-in-kubernetes-5edb2b7a5bba
---




# How to Mirror Traffic to Another Pod in Kubernetes?

Mirroring network traffic in production has a lot of benefits, such as debugging, troubleshooting, testing, monitoring, etc.
![](https://miro.medium.com/v2/resize:fit:700/0*Qxk8iDjOYsOW8AUA) Photo by  [Jovis Aloor](https://unsplash.com/@jovisjoseph?utm_source=medium&utm_medium=referral)  [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral) 
Some Kubernetes plugins allow you to  **mirror traffic**  from  **one pod**  **another** . In this tutorial, we will be using  **Istio** , a well-known Kubernetes traffic management component.
 **Istio**  provides a clear and concise  [installation document](https://istio.io/latest/docs/setup/install/istioctl/) . After installing  **istioctl**  on your computer,
Install  **istio**  with its CNI component:

```
cat <<EOF > istio-cni.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    cni:
      enabled: true
EOF
istioctl install -f istio-cni.yaml -y
```


 ***** Note: Istio CNI is required to mirror the network ***** 
Enable  [istio Sidecar](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)  in your namespace (I’ll use the default namespace):

```
kubectl label namespace default istio-injection=enabled
```


Check if Istio is operating correctly:

```
istioctl analyze
```




#  **Traffic Mirroring** 

Deploy two pods.
One will function like a production app, while the other will behave like a production app with debug mode turned on.
 *00-pod.yaml* 

```
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: payment
    type: production
  name: payment-app
spec:
  containers:
    - image: ailhan/web-debug
      name: payment-app
      imagePullPolicy: Always
      ports:
      - containerPort: 80
      env:
      - name: TEXT
        value: "Hello from the Payment App"
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: payment
    type: debug
  name: payment-app-debug
spec:
  containers:
    - image: ailhan/web-debug
      name: payment-app-debug
      imagePullPolicy: Always
      ports:
      - containerPort: 80
      env:
      - name: TEXT
        value: "Payment App - DEBUG Mode On"
```


Apply:

```
➜  ~ kubectl apply -f 00-pod.yaml
pod/payment-app created
pod/payment-app-debug created
```


 *01-service.yaml* 

```
apiVersion: v1
kind: Service
metadata:
  name: payment-app-svc
spec:
  ports:
  - port: 80
    targetPort: 80
    name: http-payment-app
  selector:
    app: payment
```


 ***** Note: A name must be added to the port The naming convention is **  ` **<protocol>[-<suffix>]** `  **** 
Apply:

```
➜  ~ kubectl apply -f 01-service.yaml
service/payment-app-svc created
```


Let’s test it:
![](https://miro.medium.com/v2/resize:fit:394/1*5ilKxbWeul0dy3RHNkR7hg.png) 
The requests are randomly distributed between two pods. It works.
For mirroring, I’ll use the  `payment-app-debug`  pod.
 *02-destination-rule.yaml* 
The destination rules define the rules that virtual services will follow.

```
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-app-destination-rule
spec:
  host: payment-app-svc
  subsets:
  - name: main
    labels:
      type: production
  - name: mirror
    labels:
      type: debug
```


Apply:

```
➜  ~ kubectl apply -f 02-destination-rule.yaml
destinationrule.networking.istio.io/payment-app-destination-rule created
```


 *03-virtual-service.yaml* 

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-app-svc-virtual-service
spec:
  hosts:
  - payment-app-svc
  http:
  - route:
    - destination:
        host: payment-app-svc
        subset: main
      weight: 100
    mirror:
      host: payment-app-svc
      subset: mirror
    mirrorPercentage:
      value: 100
```


Apply:

```
➜  ~ kubectl apply -f 03-virtual-service.yaml
virtualservice.networking.istio.io/payment-app-svc-virtual-service created
```


Let’s test it:
![](https://miro.medium.com/v2/resize:fit:290/1*ETJ0BL68zNCqWlibyIwa9Q.png) 
Istio’s virtual service intercepts the actual Service component. All requests are sent to the  `type: production`  app. Let’s examine the logs!:
Logs for the Production app:
![](https://miro.medium.com/v2/resize:fit:666/1*s7N90gmIiFgMO86v92fy_A.png) 
Logs for the Debug app:
![](https://miro.medium.com/v2/resize:fit:677/1*rZyQ49JYZQVi28MuoABgpw.png) 
It works. I got a response from the production app. However, the debug app got the same requests as well.