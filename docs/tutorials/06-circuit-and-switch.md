# Circuit and Switch
In previous sections, you have been dealing with just a single dependency. But in many
cases, you want to wait for multiple events to occur and then trigger a resource which means
you need a mechanism to determine which triggers to execute based on set of different event dependencies.
This mechanism is supported through `Circuit` and `Switch`.

<b>Note</b>: Whenever you define multiple dependencies in a sensor, the sensor applies
a `AND` operation, meaning, it will wait for all dependencies to resolve before it executes triggers.
`Circuit` and `Switch` can modify that behavior. 

## Prerequisite
Minio server must be set up in the `argo-events` namespace with a bucket called `test` and it should be available
at `minio-service.argo-events:9000`.

## Circuit
A circuit is a boolean expression. To create a circuit, you just need to define event
dependencies in groups and the sensor will apply the circuit logic on those groups.
If the logic results in `true` value, the sensor will execute the triggers else it won't.

## Switch
A switch is the conditional execution gate for a trigger.

Consider a scenario where you have a `Webhook` and `Minio` gateway, and you want
to trigger an Argo workflow if the sensor receives an event from the `Webhook` gateway,
but, another workflow if it receives an event from the `Minio` gateway.

1. Create the webhook event source and gateway. The gateway listens to HTTP requests
   on port `12000`

        kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/tutorials/05-circuit-and-switches/webhook-event-source.yaml

        kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/tutorials/05-circuit-and-switches/webhook-gateway.yaml

2. Create the minio event source and gateway. The gateway listens to events of type
   `PUT` and `DELETE` for objects in bucket `test`.

        kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/tutorials/05-circuit-and-switches/minio-event-source.yaml

        kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/tutorials/05-circuit-and-switches/minio-gateway.yaml

Make sure there are no errors in any of the gateways and all event sources are active.

3. Lets create the sensor. If you take a closer look at the trigger templates, you will
   notice that it contains `switch` key with `all` condition, meaning, execute this trigger
   when every group defined in `all` is resolved. In the sensor definition, there
   is only one group under `all` in both trigger templates. So, as soon as the group is resolved, the
   corresponding trigger will be executed.

        kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/tutorials/06-circuit-and-switches/sensor-01.yaml
   
4. Send a HTTP request to Webhook gateway,

        curl -d '{"message":"this is my first webhook"}' -H "Content-Type: application/json" -X POST http://localhost:12000/example

5. You will notice an Argo worklfow with name `group-1-xxxx` is created with following output,

         __________________________ 
        < this is my first webhook >
         -------------------------- 
            \
             \
              \     
                            ##        .            
                      ## ## ##       ==            
                   ## ## ## ##      ===            
               /""""""""""""""""___/ ===        
          ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
               \______ o          __/            
                \    \        __/             
                  \____\______/   

4. Now, lets generate a Minio event so that we can run `group-2-xxxx` workflow. Drop a file
   onto `test` bucket. The workflow that will get created will print the name of the bucket as
   follow,
   

         ______ 
        < test >
         ------ 
            \
             \
              \     
                            ##        .            
                      ## ## ##       ==            
                   ## ## ## ##      ===            
               /""""""""""""""""___/ ===        
          ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
               \______ o          __/            
                \    \        __/             
                  \____\______/   


5. Great!! You have now learned how to use a `circuit` and `switch`. Lets update the sensor with a trigger
   that waits for both groups to resolve. This is the normal sensor behavior if circuit is not defined.

        kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/tutorials/06-circuit-and-switches/sensor-02.yaml

   Send a HTTP request and perform a file drop on Minio bucket as done above. You should following output,
  

         _______________________________ 
        < this is my first webhook test >
         ------------------------------- 
            \
             \
              \     
                            ##        .            
                      ## ## ##       ==            
                   ## ## ## ##      ===            
               /""""""""""""""""___/ ===        
          ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
               \______ o          __/            
                \    \        __/             
                  \____\______/   
  
