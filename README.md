# Image Assets CDN Repository

Repository này dùng để **quản lý ảnh và folder ảnh** thông qua GitHub,  
tự động **sync lên Amazon S3** bằng **GitHub Actions**,  
và phục vụ ảnh qua **CDN (CloudFront / Cloudflare)**.

❗ Repository **KHÔNG chứa source code ứng dụng, chỉ chứa file/folder ảnh**.

---

## 🎯 Mục tiêu

- Lưu trữ ảnh bằng Git
- Khi update ảnh:
  - Tự động sync lên S3
  - Giữ nguyên cấu trúc folder
- Phục vụ ảnh qua CDN
- Không thao tác thủ công

---

## 🏗 Kiến trúc hệ thống

```
GitHub Repository (images only)
        |
        | GitHub Actions (CI/CD)
        v
      Amazon S3
        |
        v
      CDN
        |
        v
     Client / Website / App
```

---

## 📁 Cấu trúc thư mục

```text
image-assets/
├── images/
│   ├── banner/
│   │   ├── home.jpg
│   │   └── sale.png
│   ├── avatar/
│   │   └── user1.png
│   └── product/
│       ├── p1.jpg
│       └── p2.jpg
├── .github/
│   └── workflows/
│       └── sync-to-s3.yml
└── README.md
```

> **Lưu ý:** Các file ở root directory vẫn được chấp nhận, không bắt buộc phải đặt trong `image-assets/`.

---

## ☁️ Amazon S3

### Tạo S3 Bucket

1. Đăng nhập [AWS Console](https://console.aws.amazon.com/s3/)
2. Tạo bucket mới (ví dụ: `my-cdn-assets`)
3. Region: Chọn region gần người dùng nhất
4. **Block Public Access**: Tắt để cho phép CDN truy cập

### Bucket Policy (cho public read)

Bucket KHÔNG public, chỉ CloudFront được phép truy cập.

Bucket Policy (CloudFront Only – OAC)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-cdn-assets/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DISTRIBUTION_ID"
        }
      }
    }
  ]
}
```

### CORS Configuration (nếu cần)

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": []
  }
]
```

---

## 🚀 CDN Configuration

### Option 1: Amazon CloudFront

1. Tạo CloudFront Distribution
2. Origin Domain: Chọn S3 bucket vừa tạo
3. Origin Access: Public
4. Viewer Protocol Policy: Redirect HTTP to HTTPS
5. Cached HTTP Methods: GET, HEAD
6. Copy CloudFront domain name (ví dụ: `d111111abcdef8.cloudfront.net`)

**URL ảnh sau khi deploy:**

```
https://d111111abcdef8.cloudfront.net/images/banner/home.jpg
```

### Option 2: Cloudflare (với Custom Domain)

1. Add site to Cloudflare
2. Configure DNS:
   - Type: CNAME
   - Name: cdn (hoặc subdomain khác)
   - Target: `my-cdn-assets.s3.amazonaws.com`
   - Proxy status: Proxied (🧡)
3. Enable Cache trong Page Rules

**URL ảnh với custom domain:**

```
https://cdn.yourdomain.com/images/banner/home.jpg
```

---

## 🤖 GitHub Actions (CI/CD)

### 1. Thiết lập GitHub Secrets

Vào **Settings** → **Secrets and variables** → **Actions**, thêm các secrets sau:

| Secret Name             | Mô tả                    | Ví dụ                                      |
| ----------------------- | ------------------------ | ------------------------------------------ |
| `AWS_ACCESS_KEY_ID`     | AWS Access Key           | `AKIAIOSFODNN7EXAMPLE`                     |
| `AWS_SECRET_ACCESS_KEY` | AWS Secret Key           | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `AWS_REGION`            | AWS Region của S3 bucket | `us-east-1` hoặc `ap-southeast-1`          |
| `S3_BUCKET`             | Tên S3 bucket            | `my-cdn-assets`                            |

### 2. Workflow File

Tạo file `.github/workflows/sync-to-s3.yml`:

```yaml
name: Sync Images to S3

on:
  push:
    branches:
      - main
      - master
  workflow_dispatch:

jobs:
  sync-to-s3:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Sync images to S3
        run: |
          aws s3 sync . s3://${{ secrets.S3_BUCKET }}/ \
            --delete \
            --exclude ".git/*" \
            --exclude ".github/*" \
            --exclude "README.md" \
            --exclude "*.md" \
            --cache-control "public, max-age=31536000"

      - name: Invalidate CloudFront cache (optional)
        if: ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID != '' }}
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

> **Lưu ý:** Nếu dùng CloudFront và muốn invalidate cache tự động, thêm secret `CLOUDFRONT_DISTRIBUTION_ID`.

---

## 📝 Cách sử dụng

### 1. Upload ảnh mới

```bash
# Clone repo
git clone https://github.com/your-username/image-assets-cdn.git
cd image-assets-cdn

# Thêm ảnh vào folder phù hợp
cp ~/Downloads/new-banner.jpg images/banner/

# Commit và push
git add .
git commit -m "Add new banner image"
git push origin main
```

→ GitHub Actions sẽ tự động sync lên S3

### 2. Truy cập ảnh qua CDN

**CloudFront:**

```html
<img
  src="https://d111111abcdef8.cloudfront.net/images/banner/home.jpg"
  alt="Banner"
/>
```

**Cloudflare (custom domain):**

```html
<img src="https://cdn.yourdomain.com/images/banner/home.jpg" alt="Banner" />
```

**Direct S3 (không khuyến khích):**

```html
<img
  src="https://my-cdn-assets.s3.amazonaws.com/images/banner/home.jpg"
  alt="Banner"
/>
```

### 3. Xóa ảnh

```bash
# Xóa file cũ
git rm images/old-image.jpg
git commit -m "Remove old image"
git push origin main
```

→ `--delete` flag trong workflow sẽ xóa file trên S3

---

## ⚙️ Tối ưu hóa

### Nén ảnh trước khi upload

Sử dụng công cụ như:

- [TinyPNG](https://tinypng.com/)
- [ImageOptim](https://imageoptim.com/)
- [Squoosh](https://squoosh.app/)

### Cache Headers

Workflow đã thiết lập `cache-control: public, max-age=31536000` (1 năm) để tối ưu caching.

### WebP Format

Xem xét chuyển đổi sang WebP để giảm kích thước file:

```bash
# Cài đặt cwebp
brew install webp  # macOS

# Convert
cwebp -q 80 input.jpg -o output.webp
```

---

## 🔒 Bảo mật

- ✅ **KHÔNG commit** AWS credentials vào repository
- ✅ Chỉ lưu credentials trong GitHub Secrets
- ✅ Sử dụng IAM user với quyền giới hạn:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:PutObject",
          "s3:GetObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::my-cdn-assets",
          "arn:aws:s3:::my-cdn-assets/*"
        ]
      }
    ]
  }
  ```

---

## 🐛 Troubleshooting

### Workflow fails với "Access Denied"

- Kiểm tra lại AWS credentials trong GitHub Secrets
- Verify IAM user có quyền đúng

### Ảnh không load được

- Check S3 bucket policy cho phép public read
- Verify CloudFront distribution status là "Deployed"
- Check CORS configuration nếu load từ domain khác

### Cache không invalidate

- Thêm secret `CLOUDFRONT_DISTRIBUTION_ID`
- Hoặc manually invalidate trong CloudFront console

---

## 📚 Resources

- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS CLI S3 Sync](https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html)

---

## 📄 License

MIT License - Free to use for personal and commercial projects.
