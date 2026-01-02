### Why ?
Old userprofile need to be registered with CoreAPI, so that other components can talk to.
To do this, we need to seed a message to system-api-endpoints topic, coreapi will consume it and register the endpoints
### Step
1. Exec to kafka cluster pod (use K9S)
2. Run this script:
   ``` /opt/kafka/bin/kafka-console-producer.sh \
>   --bootstrap-server localhost:9092 \
>   --topic system-api-endpoints \
>   --property "parse.key=true" \
>   --property "key.separator=|" \
>   <<< 'userprofile-seed|{"ComponentName":"userprofile-seed","ReRoutes":[{"AuthenticationOptions":{"AllowedScopes":null,"AuthenticationProviderKey":"DapIdP"},"DelegatingHandlers":null,"DownstreamHostAndPorts":[{"Host":"sa-component-healthcheck-svc","Port":80}],"DownstreamPathTemplate":"/external/api/{everything}","DownstreamScheme":"http","SwaggerKey":null,"UpstreamHttpMethod":null,"UpstreamPathTemplate":"/metricscomponent/{everything}"},{"AuthenticationOptions":{"AllowedScopes":null,"AuthenticationProviderKey":"DapIdP"},"DelegatingHandlers":null,"DownstreamHostAndPorts":[{"Host":"cph-adapter-ssim-svc","Port":8080}],"DownstreamPathTemplate":"/external/api/ssim/upload","DownstreamScheme":"http","SwaggerKey":"ssim","UpstreamHttpMethod":null,"UpstreamPathTemplate":"/ssim/upload"},{"AuthenticationOptions":{"AllowedScopes":null,"AuthenticationProviderKey":"DapIdP"},"DelegatingHandlers":null,"DownstreamHostAndPorts":[{"Host":"inbound-message-api-svc","Port":8080}],"DownstreamPathTemplate":"/external/api/messages","DownstreamScheme":"http","SwaggerKey":"inbound-api","UpstreamHttpMethod":null,"UpstreamPathTemplate":"/messages"},{"AuthenticationOptions":null,"DelegatingHandlers":["ODataHandler"],"DownstreamHostAndPorts":[{"Host":"userprofiles-svc","Port":8080}],"DownstreamPathTemplate":"/odata/{everything}","DownstreamScheme":"http","SwaggerKey":"olduserprofiles","UpstreamHttpMethod":null,"UpstreamPathTemplate":"/old/userprofiles/odata/{everything}"},{"AuthenticationOptions":null,"DelegatingHandlers":null,"DownstreamHostAndPorts":[{"Host":"userprofiles-svc","Port":8080}],"DownstreamPathTemplate":"/external/{everything}","DownstreamScheme":"http","SwaggerKey":"olduserprofiles","UpstreamHttpMethod":null,"UpstreamPathTemplate":"/old/userprofiles/{everything}"}],"SwaggerEndPointOptions":[{"Config":[{"Name":"Old User Profile API","Url":"http://userprofiles-svc:8080/swagger/v1/swagger.json","Version":"v1"}],"Key":"olduserprofiles"},{"Config":[{"Name":"Atlantis Adapter API","Url":"http://sa-adapter-atlantis-svc:80/swagger/v1/swagger.json","Version":"v1"}],"Key":"atlantis"},{"Config":[{"Name":"Inbound Message API","Url":"http://inbound-message-api-svc:8080/swagger/v1/swagger.json","Version":"v1"}],"Key":"inbound-api"},{"Config":[{"Name":"SSIM API","Url":"http://cph-adapter-ssim-svc:8080/swagger/v1/swagger.json","Version":"v1"}],"Key":"ssim"}]}'
>   ```

### Step

C:/AIRHART/CPH.Local.DX/.venv/Scripts/python.exe send-kafka-message.py --topic system-api-endpoints --key userprofile-seed --message "@temp_message.json" --headers "@temp_headers.json"




