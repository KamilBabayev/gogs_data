These  3  core directories (git   gogs  ssh) creates during gogs installation and 
container creation by default. After configuring gogs as you want on your local
container you can move those 3 directories  to  26635-ATTM-EDI/docker/gogs.
These diectories will be moved inside container during deployment.

```
/data
├── conf
│   └── app.ini
├── data
│   ├── gogs.db
└── git
    └── gogs-repositories
```

I have changed  as shown below.   
```
  - name: create the VCS volumes
    with_items:
      #- /data/gogs_data/conf  
      - /data/gogs_data/gogs/conf       

  - name: copy Gogs data
    copy:
      #src: ../docker/gogs/data/
      src: ../docker/gogs/
      dest: /data/gogs_data/
  - name: build the Gogs config      (moved this part below copy gogs data)
    template:
      src: templates/app.ini.j2
      #dest: /data/gogs_data/conf/app.ini
      dest: /data/gogs_data/gogs/conf/app.ini
```
