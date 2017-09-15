VMWare Harbor "hard way"
========================

All components of harbor was separeted in directories (with a intuitive name :p);
The files have sufix, it represents:
 - dpl = Deployment
 - svc = Service
 - pvc = Persistent Volume Claim
 - secret = Secrets (passwords, certificates...)

What you NEED do to deploy
==========================
LOOK inside of all files and change what makes sense for your environment like:
 - URL's
 - Certificates (you must generate all)

Certificates
============
Take a look on script "prepare" (you can find on offline installer) specially on functions create_root_cert and create_cert (there you will find the openssl commands)

TODO
----
Create a simple way to generate certificates (a doc with copy and paste commands, works too)... you can make a PR for this :p 
