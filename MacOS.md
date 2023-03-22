```console
Install Mac Ports: sudo port install munge
#Maybe you need to add this line 'export PATH=$PATH:/opt/local/bin' in ~/.bash_profile
sudo port install munge
sudo port load munge
sudo port install slurm
```

To see what files were installed by munge, run:
port contents munge 
To later upgrade munge, run:
sudo port selfupdate && sudo port upgrade munge

To see what files were installed by slurm, run:
port contents slurm 
To later upgrade slurm, run:
sudo port selfupdate && sudo port upgrade slurm 
