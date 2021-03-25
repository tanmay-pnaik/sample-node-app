# sample node app
Sample node app for trying various openshift deployment strategies.

### S2I 
```oc new-app --name nodeapp https://github.com/ManojDhanorkar/sample-node-app --context-dir source```

### Dockerfile
```oc new-app --name nodeapp2 https://github.com/ManojDhanorkar/sample-node-app```

### Docker image
```oc new-app --name nodeapp3 thingsboard123/sample-node-app:v1```

This is qa branch
