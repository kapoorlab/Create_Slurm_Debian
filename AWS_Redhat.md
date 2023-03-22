## AWS Red Hat

```console
sudo yum install munge
sudo  /usr/sbin/create-munge-key
sudo cd /etc/munge/
cp munge.key /home
sudo chown ec2-user:ec2-user munge.key
```

(Guide A)[https://southgreenplatform.github.io/trainings/hpc/slurminstallation/]
(Guide B)[https://southgreenplatform.github.io/trainings/hpc/slurminstallation/]
