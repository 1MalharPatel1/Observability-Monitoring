# Steps

- docker build -t /application/service-a .
- docker build -t /application/service-b .
- minikube image load service-a
- minikube image load service-b