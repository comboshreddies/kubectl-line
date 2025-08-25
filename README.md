# A tool for pipe streamed kubectl execution
This tool works on top of kubectl, and uses kubectl named column output.
It is able to pass important parameteres between executions within a piped sequence.

It can be used as kubectl plugin: kubectl-line.
I link this plugin as kubectl-_ so I can use kubectl _ .
Also I link kubectl-line to a binary named _, so I can use _ directly without need to type
kubectl. This setup provides minimal typing overhead.

Whith above setup, 
``` bash
kubectl line get ns 
```
should work as same as 
``` bash
kubectl _ get ns
```
or just 
``` bash
_ get ns
```

While you could link kubectl to _ , this plugin provide ability to execute
with pipe sequence and few more features.

I use this plugin as direct kubectl extension wrapper.


# Why ?

I would usually run something like:
``` bash
kubectl --context cluster1 -n some-ns get pod --show-labels
```
and then
``` bash
LIST=$(kubectl --context clst1 -n some get pod --no-headers -l app=some| awk '{print $1 }') \
for i in $LIST ; do
   kubectl --context clst1 -n some delete pod $i;
done
```

Sometimes I would delete/restart pod, sometimes I would exec, or edit or remove or add label.
Usual setup would be to get a list of items and then to operate on list item.
One could use xargs but it won't make command line simpler.


With this tool I can:
``` bash
_ --context cluster1 -n some-ns get pod -l app=some | \
 _ delete pod {{name}}
```
or if you use it as kubectl plugin
``` bash
kubectl line --context cluser1 -n some-ns get pod -l app=some | \
 kubectl line delete pod {{name}}
```


I have a script that dumps all kubernetes objects for some cluster, but 
with this tool I can do dump with:
``` bash
_ --context minikube api-resources | \
 _ get {{kind}} -A | \
 _ get {{kind}} {{name}} -o yaml > /tmp/all_objects
```

In example above first command context field (along with result table) is being passed to second command.
Context field is then passed from second command third, along with kind (from api-resources) and namesapace field from second command output. Third command does not have to specify --context, namespace would be correctly set to a namespace of a resources listed within second command. Third command will have
kind (resource kind, like pod, service ... ) and resource name filled within {{ }} values.
left most _ get {{kind}} {{name}} so you will get list of single item objects on output

This tool will try to minimize repeating of parameters (--context, -n, --kubeconfig, or any other),
on all kubectl commands within a pipe sequence.

If you imagine how kubectl is being positioned as a command line tool, it is usually focused on exact final object/resource, weather it is a api resources list, or list of all kinds of objects, or some specific object.

One can't easily get dump of all objects, on all clusters within all kubeconfig files. It is possible
but you need to parse, chunk, iterate in bash, or any other tool.

With this tool you can get all objects more easily. If you don't wan't all of objects, if you like some subset of available resources, you can easily filter out (or filter in) those you are focused on.

If you access to a few dozens of kubernetes clusters, within multiple kube-config files,
you might need to check layout of some object on all clusters, or to check are all looking identical.

With this tool pipe approach you can, for example, dump all cluster/context objects from all kubeconfig files
``` bash
_ kc-inject ~/.kube/config ~/.kube/account1 ~/.kube/account2 ~/.kube/region-eu | \
_ config get-contexts | \
_ api-resources | \
_ get {{kind}} -A | \
_ get {{kind}} {{name}} -p yaml
```
This kubectl wrapper provides some shortcuts like api-r instead of api-resources
and cgc instead config get-contexts (to speed up typing), and few additional command for filtering.

With supported shortucts you could type same in abbrevated form:
``` bash
_ kc-inject ~/.kube/config ~/.kube/account1 ~/.kube/account2 ~/.kube/region-eu | \
_ cgc | \
_ api-r | \
_ get {{kind}} -A | \
_ get {{kind}} {{name}} -p yaml
```

With few tool specific commands you can additionally inject (extend) format
of yaml/json manifest files with specific kubernetes-config file and context attributes.
``` bash
_ kc-inject ~/.kube/config ~/.kube/account1 ~/.kube/account2 ~/.kube/region-eu | \
_ -n kube-system get pod etcd-minikube -o json | _ json_inject > /tmp/all.json
```
Now each item in output file will have extended json format with headers that look like:
``` bash
{
    "context": "minikube",
    "kubeconfig" : "/Users/none/.kube/config",
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
...
```
attribute "context" is showing origin context of item, attribute "kubeconfig" shows
file from which context is loaded.

If you do not like to extend manifest, you can put objects in correctly named files
you can always use available {{ }} tags for example:
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
Tags like {{ctx}} {{kind}}, {{name}}, if available, are replaced.

Given -n kube-system (context) and pod (kind) is silently passed via pipe stream,
so last _ get {{kind}} {{name}} is expanded as:
``` bash
kubectl --context {{ctx}} -n {{ns}} get {{kind}} {{name}}
```
and then executed with template variables being substituted for real values.


# Passing arguments between pipes

If any parameter (like kubeconfig file, context, namespace) is being
used on the previous pipe (to left) execution, it will be repeated
on all other pipe executions (to the right)

For example:
``` bash
_ --context minikube -n test-run get pods | _ get pod {{name}}
```
in second execution (_ get pod {{name}} ..) context and namespace
are passed from left side of pipe execution, so execution on right
is evaluated to
kubectl --context minikube -n test-run get pod {{name}}
and name is replaced for each item in name column that command on the left
has printed out.

Same example could be executed with kind templated:
``` bash
_ --context minikube -n test-run get pods | _ get {{kind}} {{name}}
```
Now you could execute:
``` bash
_ --context minikube -n test-run get pods,svc | _ get {{kind}} {{name}}
```
and {{kind}} will be matched.


If you want to explicitly execute some field you
can use some of {{ }} tags, explained in section below.


# Available {{ }} tags

kubeconfig file related tags:

 ?x:kc -> conditional switch kubeconfig -> --kubeconfig config

 ?:kc -> conditional kubeconfig -> config | ''

 kc -> kubeconfig -> config


context related tags:

 ?x:ctx  - conditional switch context -> --context ctx | ''

 ?:ctx - conditional context-> ctx | ''

 ctx - context -> ctx | {{ctx}}


namespace related tags:

 ?x:ns - conditional switch namespace -> -n ns | '' 

 ?:ns - conditional namespace | ''

 ns - namespace


kind tag: 

 kind -> kind

name tag:

 name -> name


Also for any named column like ROLE, AGE, NAME, NAMESPACE ... in previous pipe result
one can address those like {{ROLE}}, {{AGE}}, {{NAME}}, {{NAMESPACE}}.

This tool relies on column names, but tries to be flexible on any column name format it gets,
so as long as you keep column names with capital letters, you can use custom-columns formating of output.
For example if you specify
``` bash
-o custom-columns=ABCD:.metadata.name,XYZ:.metadata.namespace,OPQ:.kind
```
you should be able to use those column with their custom names.


# flow arangements

## startes:
Starters are those command that you can start pipe sequence with:

  _ kc-inject <file1> <file2>
    injects multiple kubectl config files in pipe stream
    ( kubectl is limited to only one config file )

  _ cgc  : shortcut for confifg get-contexts

  _ config get-contexts
    injects all avalable context from confifle

  _ api-r  : shortcut for api-resources

  _ api-resources
    shows all available resources

  _ get : list required resources

  _ top : displays required usage 


## filters:
Group of commands dedicated to filtering between pipes:

Filter in or filter out specifc column name on given regexp:
  _ + <column_name> <regex>
    include lines that match
  _ - <column_name> <regex>
    excludes lines that match


Filters out specific column with expression (gt,lt,le,ge,eq,ne,re):
  _ ? <column_name> <arg1> <operator> <arg2> 

Makes specific column values uniq (like api-resources might return duplicate resource kind):
  _ uniq <column_name>


## terminators:
Group of commands that you will execute as last in pipe sequence:

  _ get
  get can also be terminator, as it can be starter

  _ sh
   execute shell with available {{ }} tags and environment variables

  _ clean
   cleans specific plugin # tags from output

  _ yaml-inject
  _ json-inject
   inject context and kubeconfig file used to get that object
   used after kubectl -o json or -o yaml

flow terminator can be any other kubectl command (exec, logs, labels, ...)
ie command that would break kubectl pipeline execution stream, or will
not return named column results

## helpers:

  _ -help
   shows help for this tool  
  _ help
   shows help of kubectl, as this tool is kubectl plugin/wrapper

  _ examples
   show some examples

  _ env-vars
   show environment varibles that can be set to troubleshoot/debug execution of this tool

# natural flow pipe sequences

You can't combine any pipe flow you might imagine, for example
_ get | _ kc-inject
would not make sense.

Following execution pipe streams are allowed:

   kc-inject -> [ config get-contexts | api-resources | get | top | sh | ... ]

   config get-contexts -> [ api-resources | get | top | sh | ... ]

   api-resources -> [ get | top | sh | ... ]

   [ get | top ] -> [ get | top | sh | ... ]


# interesting examples to run

## getting object that do not have labels
``` bash
 _ kc-inject ~/.kube/config|_ cgc|_ api-r|_ get {{kind}} -A --show-labels | _ + LABELS '<none>' | grep -v ^#line
```

## dumping all objects for all clusters ie contexts from ~/.kube/west and ~/.kube/east file
``` bash
time /bin/bash -c "
 _ kc-inject ~/.kube/west ~/.kube/east |_ cgc|_ api-r|_ get ns|_ get {{kind}}|_ get {{kind}} {{name}} "
```


## dumping all objects for all contexts in default ~/.kube/config file

``` bash
 time /bin/bash -c "
_ cgc|_ api-r|_ get {{kind}} -A|_ get {{kind}} {{name}} "
```

## getting env from all containers within test-run namespace
``` bash
 _ -n test-run top pod --containers | _ exec {{POD}} -c {{NAME}} -- env
```

## getting all pods with not all containers ready
``` bash
_ get pod -A | _ ? READY ?1 ne ?2
```
on field READY X/Y, compares (not equal) first number (X) to second (Y) number of READY X/Y

## getting what value keys (?x) are available for filtering READY column 
``` bash
_ get pod -A | _ ? READY _? 
```
 
