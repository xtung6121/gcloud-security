# Packet Mirroring 
Tài liệu sử dụng gcloud shell. [Google Cloud gcloud Overview.](https://cloud.google.com/sdk/gcloud)
</br>

Tài liệu về packet mirroring. [Use Packet Mirroring](https://cloud.google.com/vpc/docs/using-packet-mirroring)
</br>

**Understanding Regions and Zones**

Certain Compute Engine resources live in regions or zones. A region is a specific geographical location where you can run your resources. Each region has one or more zones. For example, the us-central1 region denotes a region in the Central United States that has zones us-central1-a, us-central1-b, us-central1-c, and us-central1-f.

regions_and_zones.png

Resources that live in a zone are referred to as zonal resources. Virtual machine Instances and persistent disks live in a zone. To attach a persistent disk to a virtual machine instance, both resources must be in the same zone. Similarly, if you want to assign a static IP address to an instance, the instance must be in the same region as the static IP.

Learn more about regions and zones and see a complete list in Regions & Zones documentation.
# Overview 
Sơ đồ này cho thấy cách mà các thành phần trong Google Cloud tương tác với nhau để xử lý và quản lý lưu lượng từ Internet![image](https://github.com/user-attachments/assets/f43ff550-b25f-4dfb-9fa2-7863b06d8ece)

</br>

- **InternetHost:** đại diện cho người dùng kết nối Internet gửi yêu cầu đến hệ thống google cloud
</br>

- **WebSubnet:** là một mạng con chứa các WebServer và chịu trách nhiệm xử lý phản hồi lại InternetHost
  
</br>

- **PacketMirroringPolicy:** chính sách sao chép tập tin của Google Cloud, sao chép tất cả các lưu lượng đi vào và đi ra từ nguồn được phản chiếu (WebServer)
</br>

- **ILB_Collectors:** Bộ cân bằng tải nội bộ, đóng vai trò là một điểm thu thập dữ liệu "collector", nhận toàn bộ dữ liệu đã sao chép từ Packet Mirroring, ngoài ra Internal Load Balancing còn đảm bảo rằng các lưu lượng có thể hoạt động đồng bộ, không làm ảnh hưởng đến lưu lượng chính
</br>

- **CollectorsIDS:** Là một máy chủ khác được triển khai trên Google Compute Engine, hoạt động như một hệ thống phát hiện xâm nhập. Nó phân tích các lưu lượng đã được phản chiếu để phát hiện các mối đe dọa bảo mật, hành vi đáng ngờ hoặc các sự cố mạng.
</br>

- **Signatures:** Đại diện cho các quy tắc hoặc các mẫu tin (rules). Các quy tắc này được sử dụng để phát hiện các mối đe dọa bảo mật hoặc các sự kiện xâm nhập.
</br>

# Tóm tắt quá trình hoạt động

# Các bước thực hiện

## Bước 1: Xây dựng 2 máy chủ ảo Virtual Private Cloud (VPC) là 2 WebServer và một máy chủ ảo có vai trò là CollectorIDS
### 1. Tạo VPC networks
    gcloud compute networks create dm-stamford \
    --subnet-mode=custom

### 2. Thêm vùng dữ liệu cho mạng con VPC
    gcloud compute networks subnets create dm-stamford-REGION \
    --range=172.21.0.0/24 \
    --network=dm-stamford \
    --region=REGION
### 3. Thêm một mạng con cho Collector trong Vùng 
    gcloud compute networks subnets create dm-stamford-REGION-ids \
    --range=172.21.1.0/24 \
    --network=dm-stamford \
    --region=REGION
## Bước 2: Tạo quy tắc tường lửa và cấu hình Cloud NAT (Cloud Network Address Translation)
- **Quy tắc 1** cho phép cổng HTTP tiêu chuẩn (TCP 80) và giao thức ICMP trên tất cả các máy ảo từ tất cả các nguồn
</br>

- **Quy tắc 2** cho phép (IDS - Intrusion Detection System) nhận tất cả lưu lượng mạng từ mọi nguồn, giúp hệ thống có thể giám sát và phân tích mọi hoạt động mạng đi vào.
  
</br>

- **Quy tắc 3** cho phép dải địa chỉ IP "Google Cloud IAP Proxy" qua cổng TCP 22 tới tất cả các máy ảo (VM), cho phép SSH vào các máy ảo thông qua Cloud Console.
</br>

**Chi tiết từng câu lệnh**

    gcloud compute firewall-rules create fw-dm-stamford-allow-any-web \
    --direction=INGRESS \
    --priority=1000 \
    --network=dm-stamford \
    --action=ALLOW \
    --rules=tcp:80,icmp \
    --source-ranges=0.0.0.0/0
</br>

    gcloud compute firewall-rules create fw-dm-stamford-ids-any-any \
    --direction=INGRESS \
    --priority=1000 \
    --network=dm-stamford \
    --action=ALLOW \
    --rules=all \
    --source-ranges=0.0.0.0/0 \
    --target-tags=ids
</br>

    gcloud compute firewall-rules create fw-dm-stamford-iapproxy \
    --direction=INGRESS \
    --priority=1000 \
    --network=dm-stamford \
    --action=ALLOW \
    --rules=tcp:22,icmp \
    --source-ranges=35.235.240.0/20
</br>

**Tạo bộ định tuyến Cloud (Cloud NAT)**

    gcloud compute routers create router-stamford-nat-REGION \
    --region=REGION \
    --network=dm-stamford
</br>

**Cấu hình cho NAT**

    gcloud compute routers nats create nat-gw-dm-stamford-REGION \
    --router=router-stamford-nat-REGION \
    --router-region=REGION \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges
</br>

## Bước 3: Tạo máy ảo Virtual machines
**Chuẩn bị một máy chủ Ubuntu trong khu vực us-central1 và cài đặt một dịch vụ web đơn giản**

        gcloud compute instance-templates create template-dm-stamford-web-REGION \
        --region=REGION \
        --network=dm-stamford \
        --subnet=dm-stamford-REGION \
        --machine-type=e2-small \
        --image=ubuntu-1604-xenial-v20200807 \
        --image-project=ubuntu-os-cloud \
        --tags=webserver \
        --metadata=startup-script='#! /bin/bash
          apt-get update
          apt-get install apache2 -y
          vm_hostname="$(curl -H "Metadata-Flavor:Google" \
          http://169.254.169.254/computeMetadata/v1/instance/name)"
          echo "Page served from: $vm_hostname" | \
          tee /var/www/html/index.html
          systemctl restart apache2'
