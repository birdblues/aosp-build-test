# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu1604"
  
  # host-only 네트워크 설정 추가 (NFS 공유를 위해 필요)
  config.vm.network "private_network", ip: "192.168.56.10"

  # AOSP 빌드를 위한 리소스 할당
  config.vm.provider "libvirt" do |libvirt|
    libvirt.memory = 49152  # 메모리 크기 (48GB)
    libvirt.cpus = 28        # CPU 코어 수
    libvirt.driver = "kvm"
    libvirt.disk_bus = "virtio"
    libvirt.nic_model_type = "virtio"

    # 성능 최적화
    libvirt.cpu_mode = "host-passthrough"  # 호스트 CPU 기능 활용
    libvirt.numa_nodes = [{
      :cpus => "0-27",
      :memory => "49152"
    }]

    # 추가 디스크 설정 (buildtest_default-vdb.raw 파일명 사용)
    libvirt.storage :file,
      size: '300G',
      type: 'raw',
      bus: 'virtio',
      name: 'buildtest_default-vdb.raw',  # 파일명을 name으로 사용
      device: 'vdb'
      
    # 추가 성능 최적화 
    libvirt.disk_driver :cache => 'writeback'
    libvirt.graphics_type = "none"  # GUI 비활성
  end

  # 개선된 디스크 설정 스크립트
  config.vm.provision "disk_setup", type: "shell", run: "once", inline: <<-SHELL
    set -e  # 에러 발생 시 스크립트 중단
    
    # 디스크가 이미 마운트되어 있는지 확인
    if ! mountpoint -q /workspace; then
      echo "=== 디스크 설정 시작 ==="
      
      # 디스크 존재 확인
      if [ ! -b /dev/vdb ]; then
        echo "ERROR: /dev/vdb 디스크를 찾을 수 없습니다."
        exit 1
      fi
      
      # 디스크에 파티션이 이미 있는지 확인
      if ! fdisk -l /dev/vdb | grep -q "/dev/vdb1"; then
        echo "파티션 생성 중..."
        echo -e "n\np\n1\n\n\nw" | fdisk /dev/vdb
        
        # 파티션 테이블 재로드
        partprobe /dev/vdb
        sleep 2  # 파티션 인식 대기
      else
        echo "기존 파티션 발견: /dev/vdb1"
      fi
      
      # 파일시스템이 이미 있는지 확인
      if ! blkid /dev/vdb1 | grep -q "ext4"; then
        echo "ext4 파일시스템 생성 중..."
        mkfs.ext4 -F /dev/vdb1
      else
        echo "기존 ext4 파일시스템 발견"
      fi
      
      # 마운트 디렉토리 생성
      mkdir -p /workspace
      
      # 마운트
      echo "디스크 마운트 중..."
      mount /dev/vdb1 /workspace
      
      # fstab에 이미 있는지 확인 후 추가
      if ! grep -q "/dev/vdb1" /etc/fstab; then
        echo '/dev/vdb1 /workspace ext4 defaults 0 2' >> /etc/fstab
        echo "fstab에 영구 마운트 설정 추가"
      fi
      
      # 권한 설정
      chown vagrant:vagrant /workspace
      chmod 755 /workspace
      
      echo "=== 디스크 설정 완료 ==="
      lsblk
      df -h /workspace
    else
      echo "/workspace가 이미 마운트되어 있습니다."
      df -h /workspace
    fi
  SHELL

  # 기존 NFS 공유는 유지
  config.vm.synced_folder ".", "/vagrant", type: "nfs", mount_options: ['rw', 'vers=3', 'tcp', 'actimeo=2']

  # 나머지 파일 업로드 및 환경 설정
  config.vm.provision "file", source: "./mkimage", destination: "/tmp/mkimage"
  config.vm.provision "file", source: "./gcc.tgz", destination: "/tmp/gcc.tgz"

  # root 사용자 설정 및 AOSP 빌드 환경 구성
  config.vm.provision "shell", inline: <<-SHELL
    # root 계정 비밀번호 설정
    sudo echo 'root:vagrant' | sudo chpasswd

    # mkimage 파일 복사 및 권한 설정
    mv /tmp/mkimage /usr/bin/
    chmod +x /usr/bin/mkimage
    
    # gcc.tgz 압축 해제 및 설치
    mv /tmp/gcc.tgz /opt/
    tar -zxvf /opt/gcc.tgz -C /opt
    rm -rf /opt/gcc.tgz

    echo 'export PATH=$PATH:/opt/gcc-linaro-6.3.1-2017.02-x86_64_arm-linux-gnueabihf/bin' >> /root/.bashrc
    
    # 시스템 업데이트
    apt-get update
    apt-get upgrade -y
    
    # AOSP 빌드에 필요한 패키지 설치
    apt-get install -y wget tar gcc-arm-linux-gnueabihf
    apt-get install -y git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev libgl1-mesa-dev libxml2-utils xsltproc unzip lib32stdc++6 bc python libssl-dev
    
    # 추가 유용한 도구들
    apt-get install -y vim htop screen tmux
    
    # AOSP 빌드 환경을 위한 Java 설치
    apt-get install -y openjdk-8-jdk
    apt-get clean
        
    # 기본 환경 설정
    echo 'export USE_CCACHE=1' >> /home/vagrant/.bashrc
    echo 'export CCACHE_DIR=/root/.ccache' >> /home/vagrant/.bashrc
    echo 'export CCACHE_SIZE=50G' >> /home/vagrant/.bashrc
    
    # CCACHE 디렉토리 생성
    mkdir -p /home/vagrant/.ccache
    
    # 시스템 제한 늘리기
    echo 'fs.inotify.max_user_watches=524288' >> /etc/sysctl.conf
    sysctl -p

    # AOSP 빌드 환경 설정
    wget http://ftp.gnu.org/gnu/make/make-3.81.tar.gz
    tar zxvf make-3.81.tar.gz
    cd make-3.81
    ./configure
    make
    make install
    cd ..
    rm -rf make-3.81

  SHELL

end