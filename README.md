# A tool for streamed kubectl execution
this tool works on top of kubectl named column, being able to
pass importat parameteres between executions in pipe sequence

It can be used as kubectl plugin - kubectl-line,
or you can link it as a binary named _ , and you don't have to 
repeat kubectl line on every execution, you just type _.


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
  _ +
  _ -
  filter in or filter out specifc column name on regexp

  _ ?
  filters out specific column for some value (gt,lt,le,ge,eq,ne,re) 

  _ uniq
  makes specific column values uniq

## terminators:
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
  _ help
  _ examples
  _ env-vars


# interesting examples to run

## getting object that od not have labels
 _ kc-inject ~/.kube/config|_ cgc|_ api-r|_ get {{kind}} -A --show-labels | _ + LABELS '<none>' | grep -v ^#line

# dumping all objects from all contexts within ~/.kube/config file
time /bin/bash -c "
 _ kc-inject ~/.kube/config|_ cgc|_ api-r|_ get ns|_ get {{kind}}|_ get {{kind}} {{name}} "

or

 time /bin/bash -c "
_ kc-inject ~/.kube/config|_ cgc|_ api-r|_ get {{kind}} -A|_ get {{kind}} {{name}} "

# getting env from all containers within test-run namespace
 _ -n test-run top pod --containers | _ exec {{POD}} -c {{NAME}} -- env

