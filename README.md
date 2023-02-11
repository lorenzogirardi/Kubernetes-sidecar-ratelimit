# Kubernetes-sidecar-ratelimit

For some reasons that i will share later on in a dedicated discussion, i had the need to create a rate limit inside the application.   
<br>

For this scenario we have 2 greddy options:  
- have the logic inside the app , in a *code* way  
- have a sidecar container that has this role
<br>
<br>

## the code way  

This is the simplest one but has some side effects,  
first of all can be reaused just for the same language apps.  
Same as before , having this rate inside the app , we are creating a role inside this app  
that can be good or bad depending of the strategy applied,  
imagine the request interceptor for example will consume cpu and may *false* metrics if we are not aware of it.  
<br>
But anyway , it can be a solution , so in a simple python code you have to add the following for example  
(i will use the usual [app](https://github.com/lorenzogirardi/py-test-backend/tree/ratelimit/docker) i'm using for labs stuff)
<br>
I'ts Flask so you just have to add the following:  
in requirements.txt  --> ```Flask-Limiter```  

Inside your code , import the libraries  
```
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
```

Enable the *limiter*  
```
limiter = Limiter(
	get_remote_address,
	app=app,
	storage_uri="memory://",
)
```  

And add the limiting option in the route u'd like to rate  
```
@limiter.limit("28 per second")
```








