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
>> WebServers xử lý các yêu cầu và phản hồi lại InternetHost.
>> Đồng thời, Packet Mirroring Policy sao chép toàn bộ lưu lượng từ WebServers (cả vào và ra).
>>> Lưu lượng sao chép được chuyển đến ILB Collectors và tiếp tục đến CollectorIDS.
>>> CollectorIDS phân tích lưu lượng, so sánh với các signatures để phát hiện các mối đe dọa hoặc sự cố. 
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
   
- Cài đặt Suricata

      sudo add-apt-repository ppa:oisf/suricata-stable -y
  </br>
      
      sudo apt-get update -y
    </br>
      
      sudo apt-get install suricata -y
    </br>

- Kiểm tra Suricata
      suricata -V

## Bước 6: Cấu hình và kiểm tra Suricata

-	Các lệnh và các bước trong phần tiếp theo cũng phải được thực thi trong SSH của IDS/Suricata VM.
- Dừng dịch vụ Suricata và lưu tệp cấu hình mặc định:

      sudo systemctl stop suricata
</br>

    sudo cp /etc/suricata/suricata.yaml
    /etc/suricata/suricata.backup

- Tải xuống và thay thế tệp cấu hình Suricata mới và tệp quy tắc viết tắt
</br>

- Tệp cấu hình mới cập nhật giao diện trình thu thập và chỉ cảnh báo về một phần nhỏ lưu lượng truy cập, như được định cấu hình trong tệp my.rules và suricata.yaml. Các cấu hình mặc định của bộ quy tắc và cảnh báo Suricata vẫn giữ nguyên. Chạy các lệnh sau để sao chép các tệp:
</br>

      wget https://storage.googleapis.com/tech-academy-enablement/GCP-Packet-Mirroring-with-OpenSource-IDS/suricata.yaml
  </br>
    
      wget https://storage.googleapis.com/tech-academy-enablement/GCP-Packet-Mirroring-with-OpenSource-IDS/my.rules
  </br>

      sudo mkdir /etc/suricata/poc-rules
  </br>
    
      sudo cp my.rules /etc/suricata/poc-rules/my.rules
  </br>
    
      sudo cp suricata.yaml /etc/suricata/suricata.yaml
  </br>
    
- Bắt đầu dịch vụ Suricata

Đôi khi dịch vụ cần phải khởi động lại. Lệnh restart được đưa vào ở bước này để giải thích cho khả năng này:
</br>

        sudo systemctl start suricata
  </br>
  
        sudo systemctl restart suricata
  
- Kiểm tra bộ quy tắc cho Suricata, kết quả cho thấy tổng cộng bốn quy tắc và mô tả cho từng quy tắc:
</br>

      cat /etc/suricata/poc-rules/my.rules

## Bước 7: Cấu hình Packet Mirroring Policy

Việc thiết lập Packet Mirror Policy có thể hoàn thành bằng một lệnh đơn giản hoặc thông qua một "wizard" (trình hướng dẫn) trong GUI. Trong lệnh này, ta chỉ định 5 thuộc tính sau:
1.	Khu vực (Region)
2.	Mạng VPC (VPC Network)
3.	Nguồn cần sao chép (Mirrored Source)
4.	Bộ thu (Collector)
5.	Lưu lượng sao chép (Mirrored traffic)
Không cần chỉ định "lưu lượng sao chép" vì chính sách sẽ được cấu hình để sao chép "tất cả" lưu lượng mà không cần bộ lọc. Chính sách này sẽ sao chép cả lưu lượng vào (ingress) và ra (egress) và chuyển tiếp đến thiết bị Suricata IDS, là một phần của bộ thu ILB.
</br>

**Cấu hình Packet Mirroring Policy chạy bởi Cloud Shell**

        gcloud compute packet-mirrorings create mirror-dm-stamford-web \
        --collector-ilb=ilb-dm-stamford-suricata-ilb-REGION \
        --network=dm-stamford \
        --mirrored-subnets=dm-stamford-REGION \
        --region=REGION
        
## Bước 8: Sử dụng IDS Suricata phân tích, cảnh bảo và ngăn chặn

Tiếp theo sẽ tạo lưu lượng mạng kích hoạt từng quy tắc này. Các cảnh báo tương ứng sẽ xuất hiện trong nhật ký sự kiện Suricata.
- TEST1 và TEST2 sẽ được thực hiện từ máy chủ web và sẽ kiểm tra lưu lượng truy cập đi ra.
- TEST3 và TEST4 sẽ được khởi chạy từ Cloud Shell và kiểm tra lưu lượng truy cập vào.

  #### TEST1: Kiểm tra cảnh báo và quy tắc UDP gửi đi
-	Chạy lệnh sau từ một trong các máy chủ WEB để tạo lưu lượng DNS gửi đi:
  
        dig @8.8.8.8 example.com
 	
- Sau đó xem cảnh báo trong nhật ký sự kiện Suricata của IDS VM.
- Chuyển sang cửa sổ SSH cho VM IDS
- Chạy lệnh sau trong cửa sổ SSH của IDS VM:
  
        egrep "BAD UDP DNS" /var/log/suricata/eve.json
  
- Mục nhật ký sẽ như sau:

>> @GCP: {"timestamp":"2024-12-09T18:30:39.505175+0000","flow_id":142789263209815,"in_iface":"ens4","event_type":"alert","src_ip":"172.21.0.3","src_port":40459,"dest_ip":"8.8.8.8","dest_port":53,"proto":"UDP","alert":{"action":"allowed","gid":1,"signature_id":99996,"rev":1,"signature":"BAD UDP DNS REQUEST","category":"","severity":3},"dns":{"query":[{"type":"query","id":245,"rrname":"example.com","rrtype":"A","tx_id":0}]},"app_proto":"dns","flow":{"pkts_toserver":1,"pkts_toclient":0,"bytes_toserver":82,"bytes_toclient":0,"start":"2024-12-09T18:30:39.505175+0000"}} >>

#### TEST2: Kiểm tra quy tắc/cảnh báo TCP
- Thực hiện lệnh sau từ một trong các máy chủ WEB để tạo lưu lượng TCP đầu ra:
  
        telnet [PUBLIC_IP_WEB2] 6667
  
- Sau đó xem cảnh báo trong nhật ký sự kiện Suricata của VM IDS.
- Chuyển sang cửa sổ SSH cho VM IDS
-	Chạy lệnh sau từ cửa sổ SSH của IDS VM:
  
        egrep "BAD TCP" /var/log/suricata/eve.json
 	
- Mục nhật ký sẽ như sau:
>> @GCP: {"timestamp":"2024-12-09T18:33:25.112972+0000","flow_id":555043857480012,"in_iface":"ens4","event_type":"alert","src_ip":"172.21.0.3","src_port":52420,"dest_ip":"34.31.192.26","dest_port":6667,"proto":"TCP","alert":{"action":"allowed","gid":1,"signature_id":99999,"rev":1,"signature":"BAD TCP 6667 REQUEST","category":"","severity":3},"flow":{"pkts_toserver":1,"pkts_toclient":0,"bytes_toserver":74,"bytes_toclient":0,"start":"2024-12-09T18:33:25.112972+0000"}} 

#### TEST3: Kiểm tra quy tắc/cảnh báo ICMP đầu vào
-	Chạy lệnh sau trong Cloud Shell để tạo lưu lượng INPUT ICMP.
-	Thay thế [PUBLIC_IP_WEB1] bằng địa chỉ IP công cộng của "WEB1".
  
        ping -c 3 [PUBLIC_IP_WEB1]
 	
- Sau đó, xem cảnh báo trong nhật ký sự kiện Suricata của VM IDS:
  
        egrep "BAD ICMP" /var/log/suricata/eve.json

- Mục nhật ký sẽ như sau:

>> @GCP: {"timestamp":"2024-12-09T18:35:39.832713+0000","flow_id":2190764981073097,"in_iface":"ens4","event_type":"alert","src_ip":"35.198.215.35","src_port":0,"dest_ip":"172.21.0.3","dest_port":0,"proto":"ICMP","icmp_type":8,"icmp_code":0,"alert":{"action":"allowed","gid":1,"signature_id":99998,"rev":1,"signature":"BAD ICMP","category":"","severity":3},"flow":{"pkts_toserver":1,"pkts_toclient":0,"bytes_toserver":98,"bytes_toclient":0,"start":"2024-12-09T18:35:39.832713+0000"}}

### TEST4: Kiểm tra quy tắc/cảnh báo HTTP đầu vào
Sử dụng trình duyệt web của máy trạm cục bộ hoặc cuộn tròn trong Cloud Shell, mở địa chỉ công khai được gán cho WEB1 cho trang “index.php”, sử dụng giao thức HTTP. 
</br>

Thay thế [PUBLIC_IP_WEB1] bằng địa chỉ IP công cộng của "WEB1".

        http://[PUBLIC_IP_WEB1]/index.php
        
- Sau đó, xem cảnh báo trong nhật ký sự kiện Suricata của VM IDS:
  
        egrep "BAD HTTP" /var/log/suricata/eve.json
  
- Mục nhật ký sẽ như sau:
  
>> @GCP: {"timestamp":"2024-12-09T18:37:06.003529+0000","flow_id":1142322667531643,"in_iface":"ens4","event_type":"alert","src_ip":"1.55.81.218","src_port":5433,"dest_ip":"172.21.0.3","dest_port":80,"proto":"TCP","tx_id":0,"alert":{"action":"allowed","gid":1,"signature_id":99997,"rev":1,"signature":"BAD HTTP PHP REQUEST","category":"","severity":3},"http":{"hostname":"34.46.147.148","url":"/index.php","http_user_agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0","http_content_type":"text/html","http_method":"GET","protocol":"HTTP/1.1","status":404,"length":275},"app_proto":"http","flow":{"pkts_toserver":7,"pkts_toclient":6,"bytes_toserver":1304,"bytes_toclient":1332,"start":"2024-12-09T18:37:05.470395+0000"}} 

# Mở rộng 

## Install a NGINX web server
Now you'll install NGINX web server, one of the most popular web servers in the world, to connect your virtual machine to something.

Once SSH'ed, get root access using sudo:

      sudo su -
As the root user, update your OS:

      apt-get update
(Output)

Get:1 http://security.debian.org stretch/updates InRelease [94.3 kB]
Ign http://deb.debian.org strech InRelease
Get:2 http://deb.debian.org strech-updates InRelease [91.0 kB]
...
Install NGINX:

      apt-get install nginx -y
(Output)

Reading package lists... Done
Building dependency tree
Reading state information... Done
The following additional packages will be installed:
...
Check that NGINX is running:

      ps auwx | grep nginx

## Thống kê lưu lượng mạng


