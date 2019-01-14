## How To Duplicate an Application in K8S to Another K8S Cluster
### Requirements
We run a few applications on a Production K8S cluster. Periodically we need to duplicate Production environment to a test K8S cluster for development team.
Or We run a few applications on a test K8S cluster. Due to business requirement, we need to duplicate this application in a new K8S cluster.
We need to find a process how we can handle these requirements. In this note, we use our livesql sandbox application as an example. We plan to duplicate it  into a new K8S cluster.

### Preparation on the Source
##### Binaries
Binaries would be the easiest part. Thanks to the portability of docker images. What we need to do is to save the images to tar files and move them to the target.
* Via OCI object storage.
  * Find which host has the livesql sandbox docker images via "kubectl get po -owide"
  * login the host, find the images via "docker images|grep lsql"
  * Save the MT images to tar file via "docker save -o /u01/build/livesql-sb-v4.tar  oracle/lsqlsb:v4"
  * Use the same concept to save the DB images
  * Upload tar files to OCI object storage via OCI SDK [examples](https://github.com/HenryXie1/Examples-of-Go-Work-With-Oracle-OCI-Object-Storage) or  [OCI CLI](https://docs.cloud.oracle.com/iaas/Content/API/Concepts/cliconcepts.htm). Please refer how to config OCI CLI [link](https://docs.cloud.oracle.com/iaas/Content/API/SDKDocs/cliconfigure.htm).OCI CLI preferred for file size larger than 128M.
  ```
  #oci os object put -bn livesql-sandbox --file test.tar
  Upload ID: 0b5a080e-290e-e099-e2ab-e107a69cae83
  Split file into 3 parts for upload.
  Uploading object  [####################################]  100%
  {
    "etag": "7E9C517DA87F6DA6E053025dd20AC9AC",
    "last-modified": "Fri, 04 Jan 2019 06:37:45 GMT",
    "opc-multipart-md5": "CSnEGa2z6mD+4Yf3LVEljA==-3"
  }
  ```
 
* Via OCIR(Oracle Cloud Infrastructure Registry) which is preferred, Push or Pull the updated images from OCIR of oracle OCI. Please refer [my another note](https://www.henryxieblogs.com/2018/10/how-to-pushpull-docker-images-into.html).

```
docker login iad.ocir.io   (we use ashburn region)
Username:  <tenancyname>/test.test@oracle.com
Password:  <The Auth token you generated before>
Login succeed.
Tag the images you would like to upload
docker tag hello-world:latest
<region-code>.ocir.io/<tenancy-name>/<repo-name>/<image-name>:<tag>
docker tag hello-world:latest iad.ocir.io/***/***/hello-world:latest
Remember to add "repo-name"
Push the image to registry
docker push <region-code>.ocir.io/<tenancy-name>/<repo-name>/<image-name>:<tag>
```

##### K8S Configuration yaml files
Backup all yaml files when we first created deployments, services,statefulset, pv,pvc ...etc to object storage or local laptop
Or we can use kubectl -0 yaml to get it from Etcd
```
kubectl get svc livesqlsb-db-service -o yaml
```
##### Data
* If the target K8S cluster sits in the same OCI tenancy, Copying data would be easier as we can use backup or snapshot of block storage or file storage(NFS) to move data around. OCI even provides cross-region copy functions to help move data around the world.See [related note](https://blogs.oracle.com/cloud-infrastructure/copying-instances-or-images-across-regions)
* If the target K8S cluster sits in a different tenancy or different cloud provider, we need to use object storage or local backup to move the data.Example below are the steps how we prepare data for a different tenancy.
  * We have Oracle Database 18c K8S Pods running on OCI block storage ,same concept for file storage(NFS). Assume there is no physical standby (or no replica if it is mysql),otherwise we can use standby to prepare data
  * We prepare the DB pods creation yaml files. They are included in the above "K8S configuration yaml files" section
  * We shutdown DB via deleting deployment or statefulset...etc. Outages may take 1-10 min depends how long snapshot or backup takes. If we use standby or replica as source, it should no outages.
  * Depends where DB sits,take a snapshot of file storage. Refer [doc](https://docs.cloud.oracle.com/iaas/Content/File/Tasks/managingsnapshots.htm). Or backup of block storage. Refer [doc](https://docs.cloud.oracle.com/iaas/Content/Block/Concepts/blockvolumebackups.htm)
  * We start DB via recreating pods via yaml file we prepared.
  * Mount block storage or file storage in a temporary VM instance
  * In the temporary VM, Upload datafiles,controlfiles,archivelogs,init, tns,sqlnet ...etc files to OCI object storage via OCI SDK [examples](https://github.com/HenryXie1/Examples-of-Go-Work-With-Oracle-OCI-Object-Storage) or  [OCI CLI](https://docs.cloud.oracle.com/iaas/Content/API/Concepts/cliconcepts.htm). Please refer how to config OCI CLI [link](https://docs.cloud.oracle.com/iaas/Content/API/SDKDocs/cliconfigure.htm).OCI CLI preferred for file size larger than 128M. Use bulk-upload to upload a whole directory. Refer [doc](https://docs.cloud.oracle.com/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/os/object/bulk-upload.html)
  ```
  oci os object bulk-upload -bn livesql-sandbox --src-dir /mnt/cas-data/.snapshot/movetoesps
  ```


### Implementation on the Target
##### Binaries
* Via OCI object storage.
  * Find which host we plan to run the images
  * login the host, download tar files to OCI object storage via OCI SDK [examples](https://github.com/HenryXie1/Examples-of-Go-Work-With-Oracle-OCI-Object-Storage) or  [OCI CLI](https://docs.cloud.oracle.com/iaas/Content/API/Concepts/cliconcepts.htm). OCI CLI preferred for file size larger than 128M.
  * Use "docker load -i <path to image tar file>" to load the images into local docker registry. Then we are able to use it in yaml file.
  * Be aware of which host we load the images, it will error out if K8S try to start the image on different hosts. To avoid that,use selector to request K8S to start apps on specific hosts
* Via OCIR which is preferred. Nothing we do here, K8S will download them if the images don't exist locally.Later we start the images via
```
<region-code>.ocir.io/<tenancy-name>/<repo-name>/<image-name>:<tag>
```

##### K8S Configuration yaml files
* Download the backup files from object storage or local laptop to the host which has configurated kubectl
* Test we have correct access to OCIR
* Add correct nodeSelector label for hosts which will run DB Pods, refer [note](https://www.henryxieblogs.com/2018/12/how-to-move-existing-db-docker.html)
* Modify the yaml files to use the correct OCIR images and correct nodeSelector

##### Data
* In the target nodes which we assign to run DB pods, we mount the block storage or file storage as the exact directories as we have on the source nodes.
* Download datafiles,controlfiles,archivelogs,init, tns,sqlnet ...etc files from  OCI object storage via OCI SDK [examples](https://github.com/HenryXie1/Examples-of-Go-Work-With-Oracle-OCI-Object-Storage) or  [OCI CLI](https://docs.cloud.oracle.com/iaas/Content/API/Concepts/cliconcepts.htm). Please refer how to config OCI CLI [link](https://docs.cloud.oracle.com/iaas/Content/API/SDKDocs/cliconfigure.htm).OCI CLI preferred for file size larger than 128M.
```
oci os object bulk-download -bn livesql-sandbox --download-dir /backup/movetoesps
```
* All files should remain the same directory structure as the source as well as the file OS access permissions.
* We start DB and services via yaml files we prepared.
* Use kubectl logs <pod name> to check any error
