# A tool for pipe streamed kubectl execution
# Examples
## delete all pods in namespace prod with label app=web
``` bash
_ -n prod get pod -l app=web | _ delete pod {{name}}
```
## get all pods from all clusters in namespace prod with label app=web in all clusters
``` bash
_ config get-contexts | _ -n prod get pod -l app=web 
```
## dump all pods from prod namespace with not all containers ready
``` bash
_ -n prod get pod | _ ? READY ?1 ne ?2 | _ get pod {{name}} -o yaml
```
## show app.conf file from all clusters within east and west kubeconfig, from all clusters that match prod, from web namespace and pods labeled with app=api
``` bash
_ kci east west | _ cgc prod | _ -n web get pod -l app=api | _ exec {{name}} -- cat /app.conf
```
## show any kind of object (all kinds) with label app=web
``` bash
_ api-r | _ get {{name}} -A -l app=web
```

More examples at the bottom.

# Description

A kubectl wrapper/plugin that enables pipe-streamed execution with automatic parameter propagation.

Instead of running set of bash kubectl commands in sequence, run them in pipelined workflow.
Instead of repeating --context, -n, or resource names across multiple commands, you can build pipelined workflows where parameters and resource identifiers flow automatically.

It reduces repetitive typing and makes complex multi-step and multi-cluster kubectl operations easier to express.

# Features 

Works as a kubectl plugin (kubectl line) or standalone (_).

Behaves like kubectl when not used in a pipeline.

Run other plugins from this plugin.

Automatically passes context, namespace, kubeconfig, kind, name between piped commands.

Supports template tags ({{name}}, {{kind}}, {{ctx}}, etc.

Provides shortcuts (api-r → api-resources, cgc → config get-contexts, .

Adds filters for regex and conditional selection on columns.

Adds injectors extending manifests for better visibility.

# Installation and Setup

You can install the tool as kubectl-line.
``` bash
cp kubectl-line /usr/local/bin
```

For convenience, I symlink it to:
``` bash
ln -s kubectl-line kubectl-_
ln -s kubectl-line _
```

With above setup (ie linked kubectl-line to kubectl-_ and _ )
``` bash
kubectl line get ns 
```
should work as same as 
``` bash
kubectl _ get ns
```
or simpler 
``` bash
_ get ns
```

# Purpose of this tool ?

With vanilla kubectl I often run something like:
``` bash
kubectl --context cluster1 -n some-ns get pod --show-labels
```
observe output, and then select some subset of resources, and execute some action on those.
``` bash
LIST=$(kubectl --context clst1 -n some get pod --no-headers -l app=some| awk '{print $1 }') \
for i in $LIST ; do
   kubectl --context clst1 -n some delete pod $i;
done
```

Sometimes I would delete pod, sometimes I would exec, edit, remove or add labels.
Usual pattern of such executions is to get a list of items and then to operate on each item.


With this tool I can observe, then append line or _ after kubectl that selected resources, and add pipe:
``` bash
kubectl line -n some get pod -l app=some | \
 kubectl line delete pod {{name}}
```
or if you've linked kubectl-line to kubectl-_
```bash
kubectl _ -n some get pod -l app=some | \
kubectl _ line delete pod {{name}}
```
or if you've linked kubectl-line to _
```bash
_ -n some get pod -l app=some | \
_ delete pod {{name}}
```

In example above -n switch and value are passed from first (left side) command to second (right side of pipe)
implicitly.

# Example of a bit more advanced usage

## Dumping
For example I have a script that dumps all kubernetes objects from a cluster, but 
with this tool I can do dump with:
``` bash
_ --context mini api-resources | \
 _ get {{kind}} -A | \
 _ get {{kind}} {{name}} -o yaml \> /tmp/{{ns}}_{{kind}}_{{name}}.yaml 
```

## Executing same operation on multiple clusters 

### Single kubeconfig file

If you have multiple clusters defined within same kubeconfig file,
you can run same command on all of them:

``` bash
_ config get-contexts | \
_ -n prod get pod -l app=web
```

### Multiple kubeconfig files

You can inject multiple kubeconfig files in a pipeline:
``` bash
_ kc-inject first_kubeconfig_file second_kubeconfig_file | \
_ config get-contexts | \
_ -n prod get pod -l app=web
```

### Multiple kubeconfig files writing to local files

You might need to check the layout of same objects across multiple clusters, so you can inspect differences.
``` bash
_ kc-inject ~/.kube/west ~/.kube/east | \
_ -n kube-system get pod -l component=etcd -o json \> /tmp/{{ctx}}_pod_etcd.json
```

## Inspecting kubernetes

### Getting all resources from cluster with a label

If you like to get any object that is labeled with component=etcd then
```
_ api-resources | \
_ get {{kind}} -A -l component=etcd
```

### Dump all cluster context 
``` bash
_ kc-inject ~/.kube/europe ~/.kube/america | \
_ config get-contexts | \
_ api-resources | \
_ get {{kind}} -A | \
_ get {{kind}} {{name}} -p yaml
```

## Shortcuts

This kubectl wrapper provides some shortcuts like api-r instead of api-resources
and cgc instead config get-contexts (to speed up typing), kci instead of kc-inject
and few additional commands for filtering.

With supported shortcuts you could type same example from above in abbreviated form:
``` bash
_ kci ~/.kube/europe ~/.kube/america | \
_ cgc | \
_ api-r | \
_ get {{kind}} -A | \
_ get {{kind}} {{name}} -p yaml
```

## Filtering

### Filtering with separate pipe
If you have all contexts in single file you can filter contexts:
``` bash
_ cgc | + NAME '^.*[euro|america].*$' |\
_ get ns
```

### Internal filtering within cgc

``` bash
$ _ cgc
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
          miku       minikube   minikube   default
*         minikube   minikube   minikube   default
$ _ cgc minikube
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube   default
$ _ cgc CLUSTER minikube
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
          miku       minikube   minikube   default
*         minikube   minikube   minikube   default
$ _ cgc NAME miku
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
          miku       minikube   minikube   default
```


### Internal filtering within api-r
``` bash
$ _ api-r pods
NAME                                SHORTNAMES   APIVERSION                        NAMESPACED   KIND
pods                                po           v1                                true         Pod
pods                                             metrics.k8s.io/v1beta1            true         PodMetrics
$ _ api-r KIND Pod
NAME                                SHORTNAMES   APIVERSION                        NAMESPACED   KIND
pods                                po           v1                                true         Pod
podtemplates                                     v1                                true         PodTemplate
pods                                             metrics.k8s.io/v1beta1            true         PodMetrics
poddisruptionbudgets                pdb          policy/v1                         true         PodDisruptionBudget
$ _ api-r KIND 'Pod$'
NAME                                SHORTNAMES   APIVERSION                        NAMESPACED   KIND
pods                                po           v1                                true         Pod
```


## Injecting extented attributes

As you can dump multiple objects from multiple contexts and kubeconfig files,
there are commands that can additionally inject (extend) format
of yaml/json manifest files with specific kubernetes-config file and context attributes.

``` bash
_ kc-inject ~/.kube/config ~/.kube/account1 ~/.kube/account2 ~/.kube/region-eu | \
_ -n kube-system get pod etcd-minikube -o json | _ json_inject > /tmp/all.json
```
Now each item in output file will have extended json format with headers that look like:
``` bash
{
    "context": "minikube",
    "kubeconfig" : "/Users/none/.kube/east",
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
...
```

## Dumping to correctly named files

If you do not like to extend manifest, you can put objects in correctly named files
you can always use available {{ }} tags for example:
``` bash
_ config get-contexts | \
_ -n kube-system get pod -l k8s-app=kube-dns -o yaml \> /tmp/{{ctx}}_label_kube_dns_pods.yaml
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

## Advanced dumping 

Check example below named 'storing all content of default context in structured directory'


# How does it work - Passing arguments between pipes

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


If you want to explicitly use some named column field (kubectl output of previous pipe section) you
can use some of {{ }} tags, explained in the section below.


# Available {{ }} tags

Overall tags that begin with ? mark are conditional, if they are not empty
they will be shown, if they are empty they will be empty.
Tags that have ?x are replaced with corresponding switch, or empty
Tags without ?x: or ?: are either replaced or left as they are

## kubeconfig file related tags:

 ?x:kc -> conditional switch kubeconfig -> --kubeconfig config_file | ''

 ?:kc -> conditional kubeconfig -> config_file | ''

 kc -> kubeconfig -> config_file | {{kc}}


## context related tags:

 ?x:ctx  - conditional switch context -> --context ctx | ''

 ?:ctx - conditional context-> ctx | ''

 ctx - context -> ctx | {{ctx}}


## namespace related tags:

 ?x:ns - conditional switch namespace -> -n ns | '' 

 ?:ns - conditional namespace -> namespace | ''

 ns - namespace -> namespace | {{ns}}


## kind tag: 

 kind -> kind | {{kind}}

## name tag:

 name -> name | {{name}}

## column based dynamic tags

Also for any named column like ROLE, AGE, NAME, NAMESPACE ... in previous pipe result
(previous pipe output that is input for currently executing pipe stage)
one can address column names like {{ROLE}}, {{AGE}}, {{NAME}}, {{NAMESPACE}}.

# few more words on named columns

Keep in mind that this tool relies on column names like NAME, NAMESPACE, KIND and such,
but tries to be flexible on any column name format it gets.
As long as you keep column names with all capital letters, you can use custom-columns formatting of output.

For example if you specify
``` bash
-o custom-columns=ABCD:.metadata.name,XYZ:.metadata.namespace,OPQ:.kind
```
you should be able to use those columns (ABCD, XYZ, OPQ) with their custom names.
so for example
``` bash
... | _ sh 'echo {{ABCD}}' 
```
should work in same way as
``` bash
_ api-r KIND 'Pod$' | _ sh 'echo {{APIVERSION}}'
``` 
should work

## when you should not rely on custom column names as tags

Command @ does have special format for tags, it supports format
{{item?if_item_exists:if_item_does_not_exist}}, where item is kc, ctx, ns, kind, name
Command @ render yaml and json and there are no columns in those formats.

# Flow arrangements


## Starters (ie generators):
Starters are those commands that you can start pipe sequence with:
They either add some functionality (kc-inject) or are able to display
some information that can be used in following pipe executions.
Starters generate some content one can work with in pipe sequence.

  _ kc-inject <file1> <file2> ...
    injects multiple kubectl config files in pipe stream
    ( kubectl is limited to only one config file )

  _ kci : shortcut for kc-inject

  _ cgc  : shortcut for config get-contexts

  _ config get-contexts
    injects all available context from conf files

  _ api-r  : shortcut for api-resources

  _ api-resources
    shows all available resources

  _ get : list required resources

  _ top : displays required usage 


## Filters:

Group of commands dedicated to filtering between pipes:

Filter in or filter out specific column name on given regexp:

  _ + <column_name> <regex>
    include lines that match

  _ - <column_name> <regex>
    excludes lines that match


Filters out specific column with expression (gt,lt,le,ge,eq,ne,re):
  _ ? <column_name> <arg1> <operator> <arg2> 


Makes specific column values unique (like api-resources might return duplicate resource kind):
  _ uniq <column_name>


Commands cgc and api-r have their own internal filter so you can filter quicker.

## Terminators:

Group of commands that you will execute as last in pipe sequence.

  _ get

  get can also be terminator, as it can be starter

  _ sh

   execute shell with available {{ }} tags and environment variables

  _ clean

   cleans specific plugin # tags from input/output

  _ yaml-inject

  _ json-inject

   inject context and kubeconfig file used to get that object
   used after kubectl -o json or -o yaml

  _ @

  will work only if previous output format is yaml or json, and it will
  store file in specific templated directory



Flow terminator (last command in sequence of piped executions of this tool) can be 
any other kubectl command (exec, logs, labels, ...) ie command that would break
kubectl pipeline execution stream, or will not return named column results.

If you follow this tool communication rules, you could extend and add some
new pipe sequence compatible command. 
(for example: ... | _ sh './some_script {{KIND}}' | _ get {{kind}} | ...)


## Helpers:

  _ -help
   shows help for this tool  
  _ help
   shows help of kubectl, as this tool is kubectl plugin/wrapper

  _ examples
   show some examples

  _ env-vars
   show environment variables that can be set to troubleshoot/debug execution of this tool


# Natural flow pipe sequences

## allowed pipe flow
Following execution pipe streams are allowed:

   kc-inject -> [ config get-contexts | api-resources | get | top | sh | ... ]

   config get-contexts -> [ api-resources | get | top | sh | ... ]

   api-resources -> [ get | top | sh | ... ]

   [ get | top ] -> [ get | top | sh | ... ]

## example of wrong pipe flow
You can't combine any pipe flow you might imagine, for example
``` bash
_ get ns | _ kc-inject
```
would not make sense.


# Interesting examples to run

## getting object that do not have labels
``` bash
 _ kc-inject ~/.kube/config|_ cgc|_ api-r|_ get {{kind}} -A --show-labels | _ + LABELS '<none>'
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

## getting what rendered value keys (?x) are available for filtering READY column 
``` bash
_ get pod -A | _ ? READY _? 
```

## getting all pods that are restarted more than 3 times
``` bash
_ -n test get pod | _ ? RESTARTS ?1 gt 3
```

## getting all pods that are were restarted between 10min and 20min ago
``` bash
_ get pod -A | _ ? RESTARTS ?seconds ge 600 | _ ? RESTARTS ?seconds le 1200
```

## filtering out kubectl api-resources APIVERSION filed
``` bash
_ api-r | _ + APIVERSION k8s.io
```

## storing all content of default context in structured directory
``` bash
_ api-r | _ get {{kind}} -A | _ get {{kind}} {{name}} -o yaml |   _ @ /tmp/CLSTR/{{ns?NS:NONS}}/{{?:ns}}/{{kind}}/{{name}}.yaml
```

## running kubectl tks plugin nginx_inspect script on clusters within east and west kc files, filter in only clusters with prod in their name, and run tks plugin script on all pods labeled with app=nginx in webapp namespace
``` bash
_ kci west east | _ cgc prod | _ tks -n webapp start -l app=nginx nginx_inspect > {{k8s_pod}}.env`
```


# Requirements

kubectl
python3 (no extra deps)

