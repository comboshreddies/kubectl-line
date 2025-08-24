# A tool for pipe streamed kubectl execution
this tool works on top of kubectl named column, being able to
pass importat parameteres between executions in pipe sequence

It can be used as kubectl plugin - kubectl-line,
or you can link it as a binary named _ , and you don't have to 
repeat kubectl line on every execution, you just type _.


# Why ?

I would usually run something like:
``` bash
kubectl --context cluster1 -n some-ns get pod --show-labels
```
and then
``` for i in $(kubectl --context cluster1 -n some-ns get pod --no-headers -l app=some| awk '{print $1 }') ; do
   kubectl --context cluster1 -n some-ns delete pod $i;
done
```

Sometimes I would delete/restart pod, sometimes I would exec, or edit or remove or add label.
Usual setup would be to get a list then to operate on list item.
One could use xargs but it won't make command line simpler.

With this tool I can
``` bash
_ --context cluster1 -n some-ns get pod -l app=some | _ delete pod {{name}}
```
or if you use it as kubectl plugin
``` bash
kubectl line --context cluser1 -n some-ns get pod -l app=some | kubectl line delete pod {{name}}
```
I usually use this plugin as direct kubectl wrapper, as I link kubectl-line to /usr/local/bin/_


I would usually have a script that dumps all kubernetes objects for some cluster, but 
with this tool I can:
``` bash
_ --context minikube api-resources | _ get {{kind}} -A | _ get {{kind}} {{name}} -o yaml > all_objects
```
In example above context, namesapace field (of each get {{kind}} -A), would be passed via pipe to
left most _ get {{kind}} {{name}} so you will get list of single item objects on output


If you imagine how kubectl is being positioned as a command line tool, it is usually focused on exact final object, weather it is a api resources list, or list of all kinds of objects, or some specific object.

With this pipe approach you can for example dump all cluster/context objects from all kubeconfig files
``` bash
_ kc-inject ~/.kube/config ~/.kube/account1 ~/.kube/account2 ~/.kube/region-eu | \
_ config get-contexts | \
_ api-resources | \
_ get {{kind}} -A | \
_ get {{kind}} {{name}} -p yaml
```
This kubectl wrapper provides some shortcuts like api-r instead of api-resources
and cgc instead config get-contexts, and few additional command for filtering.

With some additional this tool specific commands you can additionally inject (extend) format
of yaml/json manifest files with specific kubernetes-config file and context attributes.
``` bash
_ kc-inject ~/.kube/config ~/.kube/account1 ~/.kube/account2 ~/.kube/region-eu | \
_ -n kube-system get pod etcd-minikube -o json | _ json_inject > /tmp/all.json
```
Now each item in output file will have extended json format
``` bash
{
    "context": "minikube",
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
...
```
attribute "context" is showing origin context of item being shown.

If you do not like to extend manifest you can always use available {{ }} tags for example
``` bash
_ config get-contexts | \
_ -n kube-system get pod -l k8s-app=kube-dns -o yaml \> /tmp/{{ctx}}_pod.yaml
```
or
``` bash
_ config get-contexts | \
_ -n kube-system get pod -l k8s-app=kube-dns | \
_ get {{kind}} {{name}}  \> /tmp/{{ctx}}_{{kind}}_{{name}}.yaml
```
Tags like {{ctx}} {{kind}}, {{name}}, if available are replaced.
Given -n kube-system (context) and pod (kind) is silently passed via pipe stream,
so last _ get {{kind}} {{name}} is executed as:
kubectl --context {{ctx}} -n {{ns}} get {{kind}} {{name}}


# Passing arguments between pipes
If any parameter (like kubeconfig file, context, namespace) is being
used on the previous pipe (to left) execution, it will be repeated
on all other pipe executions (to the right)

For example:
_ --context minikube -n test-run get pods | _ get pod {{name}}
in second execution (_ get pod {{name}} ..) context and namespace
are used (passed) from left side of pipe execution, so execution on right
is evaluated to
kubectl --context minikube -n test-run get pod {{name}}
and name is replaced for each item in name column that command on the left
has printed out.

If one that runs commands want to explicitly execute some field
it can use some of {{ }} tags, explained in sectipon below.


# Available {{ }} tags

kubeconfig file:

 ?x:kc -> conditional switch kubeconfig -> --kubeconfig config

 ?:kc -> conditional kubeconfig -> config | ''

 kc -> kubeconfig -> config


context:

 ?x:ctx  - conditional switch context-> --context ctx | ''

 ?:ctx - conditional context-> ctx | ''

 ctx - context -> ctx | {{ctx}}


namespace:

 ?x:ns - conditional switch namespace

 ?:ns - conditional namespace

 ns - namespace


kind: 

 kind -> kind

name:

 name -> name



# flow arangements

## startes:
Starters are those command that you can start pipe sequence with:

  _ kc-inject <file1> <file2>
    injects multiple kubectl config files, kubectl is limited 
    to only one config file 

  _ cgc  : shortcut for confifg get-contexts
  _ config get-contexts
    injects all avalable context from confifle

  _ api-r  : shortcut for api-resources
  _ api-resources
    shows all available resources

  _ get
  _ top
    provides required data

## filters:
Group of commands dedicated to filtering between pipes:

  _ +
  _ -
  filter in or filter out specifc column name on given regexp

  _ ?
  filters out specific column for some value (gt,lt,le,ge,eq,ne,re) 

  _ uniq
  makes specific column values uniq (like api-resources might return duplicate resource kind)

## terminators:
Group of commands that you will execute as last in pipe sequence:

  _ get
  get can also be terminator, as it can be starter
  _ sh
   execute shell with available {{.}} tags and environment variables

  _ clean
   cleans specific plugin # tags from output

  _ yaml-inject
  _ json-inject
   inject context and kubeconfig file used to get that object
   used after kubectl -o json or -o yaml

flow terminator can be any other kubectl command (exec, logs, labels, ...)
ie command that would break kubectl pipeline execution stream, or will
not return named column resutls

## helpers:

  _ -help
   shows help for this tool  
  _ help
   shows help of kubectl, as this tool is kubectl plugin/wrapper

  _ examples
   show some examples

  _ env-vars
   show environment varibles that can be set to troubleshoot/debug execution of this tool

# interesting examples to run

## getting object that od not have labels
``` bash
 _ kc-inject ~/.kube/config|_ cgc|_ api-r|_ get {{kind}} -A --show-labels | _ + LABELS '<none>' | grep -v ^#line
```

## dumping all clusters ie contexts from ~/.kube/config file
``` bash
time /bin/bash -c "
 _ kc-inject ~/.kube/config|_ cgc|_ api-r|_ get ns|_ get {{kind}}|_ get {{kind}} {{name}} "
```

or

``` bash
 time /bin/bash -c "
_ kc-inject ~/.kube/config|_ cgc|_ api-r|_ get {{kind}} -A|_ get {{kind}} {{name}} "
```

## getting env from all containers within test-run namespace
``` bash
 _ -n test-run top pod --containers | _ exec {{POD}} -c {{NAME}} -- env
```


