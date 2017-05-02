# Security related container demos.

## Image signing with RHEL
The ```atomic``` command creates an image signature by using a private gpg key to encrypt the container image manifest.

### Start a registry container w/persistent storage on the host.
```
chcon -R system_u:object_r:svirt_sandbox_file_t:s0 /var/lib/docker-registry

docker run -d -p 5000:5000 --restart always --name registry -v /var/lib/docker-registry:/var/lib/registry registry:2
```

### Generate the gpg keys. 
First, start the random data generator as this will speed up the key generation process.
```
rngd -r /dev/urandom --verbose
```

Next, generate a gpg key pair that will be used for image signing.
```
gpg --gen-key

demo2017
demo@null.net
demo
```
### Configure Docker to use signature verification.
```
less /etc/sysconfig/docker
```
### Sign and push the image.
Choose an image and tag it for the registry.
```
docker images

docker tag mystery localhost:5000/mystery

atomic push --sign-by demo2017 localhost:5000/mystery:latest
```
Looks like the push was sucessfull and a signature was created and 
stored locally. We can verify that by examing the following directory.
```
ls -R /var/lib/atomic/sigstore
```
### Trust policies.
First, let's examine the current trust policy. The default policy is to 
accept pulls from any registry.
```
atomic trust show
```
Let's change the default trust policy to reject all image pulls.
```
atomic trust default reject

atomic trust show
``` 
Now we'll try an image pull and see that it is rejected by the trust policy.
```
docker pull localhost:5000/mystery:latest
```
Next we'll create a policy to trust images signed from our registry.

First, lets export the public key so signed image manifests can be decrypted.
```
gpg --export demo2017 > /root/demo2017.pub

file /root/demo2017.pub
```
Now use the signature and the key to establish the trust.
```
atomic trust add localhost:5000 --sigstore=file:///var/lib/atomic/sigstore --pubkeys=/root/demo2017.pub

atomic trust show
```
Notice that the trust policy only accepts signed images from our registry.
Let's try the pull again and it should succeed.
```
docker pull localhost:5000/mystery:latest
```



## Container Isolation