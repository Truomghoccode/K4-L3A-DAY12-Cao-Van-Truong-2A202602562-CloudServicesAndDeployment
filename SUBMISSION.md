# Hướng Dẫn Nộp Bài

## Hình thức bài làm

Đây là **bài lab cá nhân**. Mỗi học viên phải tự nộp một repository riêng,
kể cả khi có thảo luận cách tiếp cận với học viên khác.

## Tên repository

Tên repo bắt buộc theo mẫu:
K4-L3A-DAY12-Cao-Van-Truong-2A202602562-CloudServicesAndDeployment

Quy tắc đặt tên:

- Họ tên viết liền, không dấu và không có khoảng trắng.
- Các phần được ngăn cách bằng dấu `-`.
- Mã ngày phải viết hoa đúng dạng `DAYxx`; bài này dùng `DAY12`, không dùng
  `Day12` hoặc `DAY-12`.
- Ghi đúng MSSV được cấp; không dùng nickname hoặc tài khoản GitHub thay MSSV.

Sai tên repo bị trừ điểm theo [RUBRIC.md](RUBRIC.md).

## Thành phần phải nộp

Repository nộp bài phải có tối thiểu:

- Mã nguồn trong `app/` và `utils/`.
- `Dockerfile`, `docker-compose.yml` và `.dockerignore` đã hoàn thiện.
- `exercises.md` đã trả lời đủ 10 câu bằng lời của học viên.
- `DEPLOYMENT.md` đã điền thông tin học viên, URL, platform và kết quả kiểm tra.
- Ảnh minh chứng trong `screenshots/` theo yêu cầu của `DEPLOYMENT.md`.
- Các file cấu hình deploy tương ứng với platform đã chọn.
- Bộ test và `grade.py` nguyên vẹn để Lab Coach có thể chấm lại.

Không nộp `.env`, API key, token, mật khẩu, private key hoặc dữ liệu nhạy cảm.

## Nơi nộp và quyền truy cập

Nộp **link repository GitHub** lên Codelab. Repo phải ở chế độ public để Lab
Coach truy cập được trong thời gian chấm.

## Kiểm tra trước khi nộp

```bash
pytest tests/ -v
python grade.py
git status --short
git ls-files | grep -E '(^|/)\.env$|\.(pem|key)$'
```

Trước khi gửi link, xác nhận:

- Repo đúng tên và có thông tin nhận diện học viên.
- Không còn `NotImplementedError` trong `app/`.
- Đã biết rõ test nào pass, test nào fail và nguyên nhân.
- `DEPLOYMENT.md` không còn placeholder và không chứa giá trị secret.
- Có lịch sử commit thể hiện tiến trình làm bài.
