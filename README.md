# Redroid Deploy (rk3588 armbian vender kernel fix)

## 1. Preparation
`sudo nano /usr/local/bin/redroid-init.sh`
```bash
#!/bin/bash

# --- 1. BinderFS  ---
mkdir -p /dev/binderfs

if ! mountpoint -q /dev/binderfs; then
    mount -t binder binder /dev/binderfs
fi

chmod 666 /dev/binderfs/binder
chmod 666 /dev/binderfs/hwbinder
chmod 666 /dev/binderfs/vndbinder

# --- 2. DMA Heap Fix (Out of Memory / No such file) ---
if [ -e /dev/dma_heap/reserved ]; then
    chmod 666 /dev/dma_heap/reserved
    
    # If you already have linux,cma and system-uncached-dma32, comment following lines
    # -------------
    rm -f /dev/dma_heap/linux,cma
    ln -s reserved /dev/dma_heap/linux,cma
    
    rm -f /dev/dma_heap/system-uncached-dma32
    ln -s reserved /dev/dma_heap/system-uncached-dma32
    # -------------
    
    chmod 666 /dev/dma_heap/*
fi

echo "Redroid environment initialized successfully."
```

`sudo chmod +x /usr/local/bin/redroid-init.sh`
`sudo nano /etc/systemd/system/redroid-init.service`

```bash
[Unit]
Description=Initialize Redroid Environment (BinderFS & DMA Heaps)
Before=docker.service
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/redroid-init.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
`sudo systemctl daemon-reload`
`sudo systemctl enable redroid-init.service`
`sudo systemctl start redroid-init.service`

## 2. Docker Compose Config
```yaml
services:
  redroid:
    #image: redroid/redroid:14.0.0-latest
    #image: iceblacktea/redroid-arm64:12.0.0-250116
    image: cnflysky/redroid-rk3588:lineage-20
    container_name: redroid
    privileged: true
    restart: unless-stopped
    
    ports:
      - "5555:5555"
        #- "50100:50000-50100/udp"
    volumes:
      - ./data:/data
      - /home/hiroi/Shared:/shared
      
      - /dev/dma_heap:/dev/dma_heap
      - /dev/binderfs:/dev/binderfs
      - /dev/binderfs/binder:/dev/binder
      - /dev/binderfs/hwbinder:/dev/hwbinder
      - /dev/binderfs/vndbinder:/dev/vndbinder
    devices: 
      - /dev/mali0:/dev/mali0
      - /dev/mpp_service:/dev/mpp_service
      - /dev/rga:/dev/rga
    command:
      - androidboot.redroid_gpu_mode=mali

      - androidboot.redroid_width=2560
      - androidboot.redroid_height=1440
      - androidboot.redroid_fps=60
      - ro.build.characteristics=tablet
      #- androidboot.redroid_magisk=1
      - androidboot.use_memfd=true
    networks:
      - redroid-net
  webscreen:
    image: dukihiroi/webscreen:latest
    container_name: webscreen
    privileged: true
    network_mode: "host"


networks:
  redroid-net:
    driver: bridge

```

## 3. Enjoy!
```bash
docker compose up -d
```
Then visit `<your server ip>:8079`, and connect `127.0.0.1:5555`
