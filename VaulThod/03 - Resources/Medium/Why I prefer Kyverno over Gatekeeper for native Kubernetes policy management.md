---
tags:
  - APP/KYVERNO
source: https://medium.com/@glen.yu/why-i-prefer-kyverno-over-gatekeeper-for-native-kubernetes-policy-management-35a05bb94964
---
## I used to use  [Open Policy Agent Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/)  for Kubernetes policies but personally found writing new policies to be quite difficult with a steep learning curve. I then decided to give  [Kyverno](https://kyverno.io/)  a try and was really impressed with how easy it was to use.

![](https://miro.medium.com/v2/resize:fit:700/1*U4vnl2qoXBNzOFeMrkR87A.png) 
Let’s jump straight into it! To compare the two policy engines, let’s start with a simple example. I will use the “hello world” of Kubernetes policies: a namespace label requirement. I will implement a policy which require new namespaces to have an  *owner*  and  *env*  label. I will be then attempting to create a namespace using the following manifest:
 **NOTE:**  the last line is commented out to trigger the desired error


# Gatekeeper

Gatekeeper policies consists of  ** *two components* ** : a constraint template and the specific constraint. The constraint template is usually generic because even after defining the constraint, it must still be applied to resources. In the following example, I am defining a constraint template that enforces the presence of a label on resources:
As mentioned previously, solely having the constraint template is not sufficient — you need to apply template to a resource. The manifest below applies a constraint using the rules set by the constraint template and applies it to namespaces only:
An attempt to apply the namespace manifest will produce the following error message:

```
Error from server (Forbidden): error when creating "./namespace.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [require-ns-labels] you must provide labels: {"env"}
```


It is a pretty clear message spelling out what was denied, the admissions controller that denied it, the rule and the fix required to make it compliant.


## Rego

At the heart of this constraint template is  [Rego](https://www.openpolicyagent.org/docs/latest/#rego) , a purpose-built language for expressing policies. This is where most of the policy magic happens, but unfortunately it is a whole new language to learn just for writing policies. The good folks at OPA do maintain and provide a  [helpful library](https://github.com/open-policy-agent/gatekeeper-library)  of constraint templates, but I do still find it a little difficult to understand sometimes. For me, this was the main reason why I was never a big fan of Gatekeeper.


# Kyverno

Reading and writing policies with Kyverno feels very natural because it is resembles  *selector*  statements found in many Kubernetes deployment manifests. Advanced uses of Kyverno will require more in-depth knowledge, but by and large, I found Kyverno very easy to pick up.
The following is Kyverno’s implementation of the namespace label requirement:
 **NOTE:**  here, I added restrictions on the values that the  *env*  label can be set to that I was not able to do in Gatekeeper
Like Gatekeeper, Kyverno also produces a clear error message outlining what was blocked, what performed the blocking, the policy and rule names, and the violation:

```
Error from server: error when creating "namespace.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Namespace//unicorn-app was blocked due to the following policies

require-ns-label:
  require-ns-env-label: 'validation error: You must have label `env` with a value
    of `dev`, `stage`, or `prod` set on all new namespaces. rule require-ns-env-label
    failed at path /metadata/labels/env/'
```




# I am convinced! I am making the switch now!

Not so fast! While Kyverno is more user friendly, it is only for Kubernetes. OPA Gatekeeper, on the other hand, has been around much longer and can extend its capabilities beyond just Kubernetes. For instance, Google Cloud’s Anthos  [Policy Controller](https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller)  is based on Gatekeeper, as well as  [Forseti](https://forsetisecurity.org/)  (though now sunset/archived). The choice between the two depends on your existing tooling for managing organizational policies and the platforms you currently use or intend to adopt.


# Conclusion

Kyverno and Gatekeeper are both robust policy engines, sharing many similar features and use cases. Yet, Kyverno stands out for the straightforwardness in its policy composition. Personally, ease of use holds considerable weight when selecting a tool, making Kyverno particularly appealing to me.