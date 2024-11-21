Scaling [--solutionname--] With Kubernetes
===========================

You can scale your solution with Kubernetes.  To do so, will will need to apply the following YAML files to your Kubernetes cluster.

.. tip::
   Refer to TML documentation for more information on `scaling with Kubernetes <https://tml.readthedocs.io/en/latest/kube.html>`_.

   You can also run the YAML files locally in a 1 node kubernetes cluster called minikube.  Refer to `Installing minikube <https://tml.readthedocs.io/en/latest/kube.html#installing-minikube>`_

.. important:: 
   Below assumes you have a Kubernetes cluster and **kubectl** installed in your Linux environment.

Based on your TML solution [--solutionname--] - if you want to scale your application with Kubernetes - you will need to apply the following YAML files.

.. list-table::

   * - **YML File**
     - **Description**
   * - :ref:`--solutionnamefile--`
     - This is your main solution YAML file.  
 
       It MUST be applied to your Kubernetes cluster.
   * - :ref:`mysql-storage.yml`
     - This is storage allocation for MySQL DB.
 
       It MUST be applied to your Kubernetes cluster.
   * - :ref:`mysql-db-deployment.yml`
     - This is the MySQL deployment YAML.
 
       It MUST be applied to your Kubernetes cluster.
   * - :ref:`privategpt.yml`
     - This is the privateGPT deployment YAML.
 
       This is OPTIONAL.  However, it must be 
 
       applied if using Step 9 DAG.
   * - :ref:`qdrant.yml`
     - This is the Qdrant deployment YAML.
 
       This is OPTIONAL.  However, it must be 
 
       applied if using Step 9 DAG and

       WEB_CONCURRENCY > 1, in the privategpt.yml.

kubectl apply command
-----------------

.. important::
   To apply the YAML files below to your Kubernetes cluster simply run this command:

.. code-block:: YAML

   --kubectl--

--solutionnamefile--
------------------------

.. important::
   Copy and Paste this YAML file: --solutionnamefile-- - and save it locally.

.. code-block:: YAML

   --solutionnamecode--

mysql-storage.yml
------------------------

.. important::
   Copy and Paste this YAML file: mysql-storage.yml - and save it locally.

.. code-block:: YAML

      apiVersion: v1
      kind: PersistentVolume
      metadata:
        name: mysql-pv-volume
        labels:
          type: local
      spec:
        storageClassName: manual
        capacity:
          storage: 20Gi
        accessModes:
          - ReadWriteOnce
        hostPath:
          path: "/mnt/data"
      ---
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: mysql-pv-claim
      spec:
        storageClassName: manual
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi

mysql-db-deployment.yml
------------------------

.. important::
   Copy and Paste this YAML file: mysql-db-deployment.yml - and save it locally.

.. code-block:: YAML

      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: mysql
      spec:
        selector:
          matchLabels:
            app: mysql
        strategy:
          type: Recreate
        template:
          metadata:
            labels:
              app: mysql
          spec:
            containers:
            - image: maadsdocker/mysql:latest
              name: mysql
              env:
              - name: MYSQL_ROOT_PASSWORD
                value: "raspberry"
              - name: MYSQLDB
                value: "tmlids"
              - name: MYSQLDRIVERNAME
                value: "mysql"
              - name: MYSQLHOSTNAME
                value: "mysql:3306"
              - name: MYSQLMAXCONN
                value: "4"
              - name: MYSQLMAXIDLE
                value: "10"
              - name: MYSQLPASS
                value: "raspberry"
              - name: MYSQLUSER
                value: "root"                  
              ports:
              - containerPort: 3306
                name: mysql
              volumeMounts:
              - name: mysql-persistent-storage
                mountPath: /var/lib/mysql
            volumes:
            - name: mysql-persistent-storage
              persistentVolumeClaim:
                claimName: mysql-pv-claim
      
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: mysql
      spec:
        ports:
        - port: 3306
        selector:
          app: mysql

privategpt.yml (Optional If Step 9 Dag is used)
---------------

.. important::
   Copy and Paste this YAML file: privategpt.yml - and save it locally.

.. note::
   By default this assumes you have a Nvidia GPU in your machine and so it using the Nvidia privateGPT container:

    **image: maadsdocker/tml-privategpt-with-gpu-nvidia-amd64**

   if you DO NOT have a Nvidia GPU installed then change image to:

    **image: maadsdocker/tml-privategpt-no-gpu-amd64**

.. code-block:: YAML

      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: privategpt
      spec:
        selector:
          matchLabels:
            app: privategpt
        replicas: 1 # tells deployment to run 1 pods matching the template
        template:
          metadata:
            labels:
              app: privategpt
          spec:
            #hostNetwork: true
            containers:
            - name: privategpt
              image: maadsdocker/tml-privategpt-with-gpu-nvidia-amd64 # IF you DO NOT have NVIDIA GPU use: maadsdocker/tml-privategpt-no-gpu-amd64
             # resources:
              #  limits:
               #   nvidia.com/gpu: 1 # requesting 1 GPU        
              ports:   
              - containerPort: 8001
              env:
              - name: NVIDIA_VISIBLE_DEVICES 
                value: all
              - name: DP_DISABLE_HEALTHCHECKS
                value: xids
              - name: WEB_CONCURRENCY
                value: "1"
              - name: GPU
                value: "1"          
              - name: COLLECTION
                value: "tml"  
              - name: PORT
                value: "8001"  
              - name: CUDA_VISIBLE_DEVICES
                value: "0"  
               
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: privategpt
        labels:
          app: privategpt
      spec:
        type: NodePort #Exposes the service as a node ports
        ports:
        - port: 8001
          name: p1
          protocol: TCP
          targetPort: 8001
        selector:
          app: privategpt
          
qdrant.yml (Optional If Step 9 Dag is used and WEB_CONCURRENCY > 1 in privategpt.yml)
---------------

.. important::
   Copy and Paste this YAML file: qdrant.yml - and save it locally.

.. code-block:: YAML

      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: qdrant
      spec:
        selector:
          matchLabels:
            app: qdrant
        replicas: 1 
        template:
          metadata:
            labels:
              app: qdrant
          spec:
            #hostNetwork: true
            containers:
            - name: qdrant
              image: qdrant/qdrant 
              ports:   
              - containerPort: 6333
              volumeMounts:
              - mountPath: /qdrant/storage
                name: qdata
            volumes:
            - name: qdata
              hostPath:
                path: /qdrant_storage          
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: qdrant
        labels:
          app: qdrant
      spec:
        type: NodePort #Exposes the service as a node ports
        ports:
        - port: 6333
          name: p1
          protocol: TCP
          targetPort: 6333
        selector:
          app: qdrant
          
