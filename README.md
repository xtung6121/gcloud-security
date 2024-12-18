# Packet Mirroring 
Các câu lệnh sử dụng gcloud shell. [Google Cloud gcloud Overview.](https://cloud.google.com/sdk/gcloud)
</br>
Tài liệu về packet mirroring. [Use Packet Mirroring](https://cloud.google.com/vpc/docs/using-packet-mirroring)
# Overview 
Sơ đồ này cho thấy cách mà các thành phần trong Google Cloud tương tác với nhau để xử lý và quản lý lưu lượng từ Internet![image](https://github.com/user-attachments/assets/f43ff550-b25f-4dfb-9fa2-7863b06d8ece)

</br>
InternetHost: đại diện cho người dùng kết nối mạng là máy chủ
</br>
WebServers: được tạo ra bởi Google cloud đóng vai trò là VM
</br>
PacketMirroringPolicy: chính sách sao chép tập tin của Google Cloud
</br>
ILB_Collectors: Bộ cân bằng tải nội bộ, thu thập các lưu lượng mạng và phân chia các lưu lượng mạng đối với mỗi VM
</br>
CollectorsIDS: Có nhiệm vụ bắt các gói tin và gửi đến thông báo 
</br>
Signatures: Các luật giúp đưa ra cảnh báo
</br>

# Tóm tắt quá trình hoạt động

# Các bước thực hiện

## Bước 1: Tạo VPC networks
    gcloud compute networks create dm-stamford \
    --subnet-mode=custom

  ### Thêm vùng dữ liệu cho mạng con VPC
    gcloud compute networks subnets create dm-stamford-REGION \
    --range=172.21.0.0/24 \
    --network=dm-stamford \
    --region=REGION
    
