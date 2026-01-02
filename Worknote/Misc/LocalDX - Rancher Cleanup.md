>[! info] Remember to check dependency
>Check for rancher desktop k8s version, traefik must be disabled

### Clean up disk 
1. Exec wsl 
2. > docker system prune -a --volumes
3. vdisk 1 : 
   "C:\Users\nvtk\AppData\Local\rancher-desktop\distro-data\ext4.vhdx"
4. vdisk 2: "C:\Users\nvtk\AppData\Local\Packages\Canoni......\LocalState\ext4.vhdx"
5. > wsl --shutdown
6. > Optimize-VHD -Path "C:\Users\nvtk\AppData\Local\rancher-desktop\distro-data\ext4.vhdx" -Mode Full
7. > diskpart
8. > select vdisk file="path"
9. > compact vdisk
10. > powercfg.exe /hibernate off (Hibernation file takes ~20gigs)

