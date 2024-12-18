# Packet Mirroring 
Tài liệu sử dụng gcloud shell. [Google Cloud gcloud Overview.](https://cloud.google.com/sdk/gcloud)
</br>

Tài liệu về packet mirroring. [Use Packet Mirroring](https://cloud.google.com/vpc/docs/using-packet-mirroring)
</br>

**Hiểu về Regions and Zones**

Tài nguyên tồn tại trong một vùng được gọi là tài nguyên theo vùng. Các phiên bản máy ảo và đĩa cứng không thay đổi tồn tại trong một vùng. Để gắn một đĩa cứng không thay đổi vào một phiên bản máy ảo, cả hai tài nguyên phải nằm trong cùng một vùng. Tương tự, nếu muốn gán một địa chỉ IP tĩnh cho một phiên bản, phiên bản đó phải nằm trong cùng khu vực với địa chỉ IP tĩnh.

Các ví dụ về REGION và ZONES: us-central1-a, us-central1-b, us-central1-c, and us-central1-f.

Chi tiết về Regions và zones. [Regions & Zones documentation](https://cloud.google.com/compute/docs/regions-zones)

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
> InternetHost gửi yêu cầu tới WebServers.
> WebServers xử lý các yêu cầu và phản hồi lại InternetHost.
> Đồng thời, Packet Mirroring Policy sao chép toàn bộ lưu lượng từ WebServers (cả vào và ra).
> Lưu lượng sao chép được chuyển đến ILB Collectors và tiếp tục đến CollectorIDS.
> CollectorIDS phân tích lưu lượng, so sánh với các signatures để phát hiện các mối đe dọa hoặc sự cố. 
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
**Chuẩn bị một máy chủ Ubuntu trong khu vực REGION (tùy chọn) và cài đặt một dịch vụ web đơn giản**

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

  **Tạo một nhóm phiên bản được quản lý cho các máy chủ web, lệnh này sử dụng mẫu phiên bản từ bước trước để tạo hai máy chủ web**

      gcloud compute instance-groups managed create mig-dm-stamford-web-REGION \
        --template=template-dm-stamford-web-REGION \
        --size=2 \
        --zone="ZONE"

#### Tạo một phiên bản cho IDS VM
- Chuẩn bị một máy chủ Ubuntu ở khu vực REGION mà không có địa chỉ IP công cộng
  
       gcloud compute instance-templates create template-dm-stamford-ids-REGION \
        --region=REGION \
        --network=dm-stamford \
        --no-address \
        --subnet=dm-stamford-REGION-ids \
        --image=ubuntu-1604-xenial-v20200807 \
        --image-project=ubuntu-os-cloud \
        --tags=ids,webserver \
        --metadata=startup-script='#! /bin/bash
          apt-get update
          apt-get install apache2 -y
          vm_hostname="$(curl -H "Metadata-Flavor:Google" \
          http://169.254.169.254/computeMetadata/v1/instance/name)"
          echo "Page served from: $vm_hostname" | \
          tee /var/www/html/index.html
          systemctl restart apache2'
#### Tạo một nhóm phiên bản được quản lý cho IDS VM
- Lệnh này sử dụng mẫu phiên bản từ bước trước để tạo VM sẽ được định cấu hình thành IDS. Cài đặt Suricata sẽ được đề cập trong phần sau.

        gcloud compute instance-groups managed create mig-dm-stamford-ids-REGION \
          --template=template-dm-stamford-ids-REGION \
          --size=1 \
          --zone="ZONE"

## Bước 4: Tạo cân bằng tải nội bộ (Internal Load Balancer - ILBCollector)

  ### 1. Ánh xạ gói tin sử dụng bộ cân bằng tải nội bộ (ILB) để chuyển tiếp lưu lượng được phản chiếu tới collector. Trong trường hợp này, collector chứa một máy ảo
     
         gcloud compute health-checks create tcp hc-tcp-80 --port 80

  ### 2. Tạo một nhóm dịch vụ phụ trợ để sử dụng cho ILB:

        gcloud compute backend-services create be-dm-stamford-suricata-REGION \
        --load-balancing-scheme=INTERNAL \
        --health-checks=hc-tcp-80 \
        --network=dm-stamford \
        --protocol=TCP \
        --region=REGION
### 3. Thêm nhóm phiên bản do IDS quản lý đã được tạo vào nhóm dịch vụ phụ trợ đã tạo ở bước trước:

        gcloud compute backend-services add-backend be-dm-stamford-suricata-REGION \
        --instance-group=mig-dm-stamford-ids-REGION \
        --instance-group-zone="ZONE" \
        --region=REGION
### 4. Tạo quy tắc chuyển tiếp giao diện để đóng vai trò là điểm cuối bộ sưu tập:

       gcloud compute forwarding-rules create ilb-dm-stamford-suricata-ilb-REGION \
       --load-balancing-scheme=INTERNAL \
       --backend-service be-dm-stamford-suricata-REGION \
       --is-mirroring-collector \
       --network=dm-stamford \
       --region=REGION \
       --subnet=dm-stamford-REGION-ids \
       --ip-protocol=TCP \
       --ports=all
## Bước 5: Cài đặt và cấu hình phần mềm mã nguồn mở IDS-Suricata
- Kết nối trong SSH với VM IDS
- Từ Bảng điều khiển đám mây, trong menu điều hướng, điều hướng đến Compute Engine > VM Instances.

![image](https://github.com/user-attachments/assets/d07d533e-2c99-4248-88d6-f1a8684f6974)
</br>

![image](https://github.com/user-attachments/assets/2ee2afb4-b2b5-45bd-9435-11533838afd3)

- Một cửa sổ mới mở ra, cho phép ta chạy các lệnh trong IDS VM. Cập nhật VM IDS:

      sudo apt-get update -y
  
- Cài đặt các gói dữ liệu phụ thuộc Suricata:

      sudo apt-get install libpcre3-dbg libpcre3-dev autoconf automake libtool libpcap-dev libnet1-dev libyaml-dev zlib1g-dev libcap-ng-dev libmagic-dev libjansson-dev libjansson4 -y
  </br>
      
      sudo apt-get install libnspr4-dev -y
   </br>
      
      sudo apt-get install libnss3-dev -y
   </br>
      
      sudo apt-get install liblz4-dev -y
   </br>
      
      sudo apt install rustc cargo -y
   </br>
