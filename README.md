# Face Recognition Service API Documentation

## 1. Kiến Trúc Hệ Thống

Hệ thống được chia thành các phần chính:

- **Config:** Quản lý đượng dận model và file database.

- **Models:** Định nghĩa schema cho request/response (Sử dụng Pydantic).

- **Utils:** Các hàm xử lý ảnh (embedding, crop, lưu ảnh, tạo URL).

- **Services:**

  - `faceDetectionService`: Sử dụng Retina Face phát hiện khuôn mặt.

  - `faceRecognitionService`: Sử dụng Arcface tính toán embedding khuôn mặt.

  - `faceDatabaseService`: Quản lý embedding bằng Faiss và thông tin người dùng (metadata).

- **Routers (API Endpoints):**

  - **/detect-faces/**: Detect khuôn mặt và lưu ảnh crop face theo `message_id`.

  - **/person/add-face/**: Thêm (hoặc cập nhật) nguời dùng dựa trên những ảnh đã chọn.

  - **/infer/**: Nhận dạng khuôn mặt từ ảnh crop person.

  - **/persons/**: Lấy danh sách người trong database (và DELETE /person/{person_uuid} để xóa).

- **Static Files:**

  - Ảnh crop tạm thời được lưu tại `data/temp/<message_id>/`.

  - Ảnh của mỗi người dùng được lưu tại `data/persons/<person_uuid>_<name>/`.

- **Persistence:**
  - Embedding và metadata được lưu trữ qua FAISS index (`data/faces.index`) và file `data/metadata.npy`.

## 2. Cấu Trúc Thư Mục

```plaintext
face_recognition
Cấu trúc thư mục:

FR/
  data/
   |   │ temp/ (# ảnh lưu tạm thời theo message_id)
   |   │ persons/ (# ảnh người)
  face_db/
   |   | faces.index   (# FAISS index)
   |   | metadata.npy   (# Metadata)
  src/
   | models/
   |   │ schemas.py
   | routers/
   |   │ faceDetectionRouter.py
   |   │ addFaceRouter.py
   |   │ personRouter.py
   |   │ faceInferenceRouter.py
   |   │ faceFilterRouter.py
   | services/
   |   │ faceDetectionService.py
   |   │ faceRecognitionService.py
   |   │ faceDatabaseService.py
   | utils/
   |   │ __init__.py
   |   │ imageProcessingUtils.py
   | config.py
   | main.py
  test/
   |   | frontend.py
  README.md
```

## 3. Chi tiết API

### 3.1. List Persons API

-   **Tóm tắt:** Lấy danh sách tất cả người trong cơ sở dữ liệu.

-   **Mô tả:** Endpoint này truy xuất thông tin về tất cả người dùng hiện có trong cơ sở dữ liệu nhận dạng khuôn mặt. Thông tin này bao gồm ID người dùng và trạng thái tồn tại của thư mục ảnh khuôn mặt liên quan.

-   **Phương thức:** `GET`

-   **Endpoint URL:** `/persons/`

-   **Tham số:** Không có.

-   **Request Body:** Không có.

-   **Response:**

    -   **`200 OK` (Thành công):** Trả về một danh sách JSON chứa thông tin của mỗi người dùng.

        ```json
        [
          {
            "user_id": 12345,
            "exists": true
          },
          {
            "user_id": 67890,
            "exists": false
          }
        ]

        ```

        -   `user_id` (integer): ID duy nhất của người dùng.
        -   `exists` (boolean): Cho biết thư mục chứa ảnh khuôn mặt của người dùng có tồn tại hay không. `true` nếu thư mục tồn tại, `false` nếu không.

-   **Xử lý:**

    1.  Truy xuất metadata từ `face_database`.
    2.  Duyệt qua từng mục trong metadata.
    3.  Kiểm tra sự tồn tại của thư mục ảnh khuôn mặt tương ứng với `user_id`.
    4.  Trả về danh sách các đối tượng `PersonResponse` chứa `user_id` và `exists`.

### 3.2. Detect Faces API

-   **Tóm tắt:** Phát hiện khuôn mặt từ danh sách các URL ảnh và trả về kết quả tải lên MinIO.

-   **Mô tả:** Endpoint này nhận một danh sách các URL ảnh, tải ảnh về, phát hiện khuôn mặt trong mỗi ảnh, cắt các khuôn mặt đã phát hiện, tải các khuôn mặt đã cắt lên MinIO và trả về danh sách các URL của các ảnh khuôn mặt đã tải lên.

-   **Phương thức:** `POST`

-   **Endpoint URL:** `/detect-faces/`

-   **Tham số:**

    -   **Request Body (JSON):**

        ```json
        {
          "image_urls": [
            "url_anh_1",
            "url_anh_2",
            ...
          ]
        }

        ```

        -   `image_urls` (list of strings, required): Danh sách các URL của ảnh cần phát hiện khuôn mặt.
    -   **Query Parameters:**

        -   `message_id` (string, optional): ID tin nhắn tùy chọn (UUID) để nhóm các ảnh khuôn mặt đã phát hiện. Nếu không được cung cấp, một UUID mới sẽ được tạo.
-   **Request Body:**

    -   `image_urls`: Danh sách các URL ảnh để phát hiện khuôn mặt.
-   **Response:**

    -   **`200 OK` (Thành công):** Trả về một JSON chứa thông tin về kết quả tải lên MinIO.

        ```json
        {
          "bucket": "q100-face-recogs",
          "datas": "datas": [
            {
                "object_name": "detectface/e029ef8c-ee59-4ae7-be96-690e96dc72a0.jpg",
                "url": "http://103.176.147.25:8004/q100-face-recogs/detectface/e029ef8c-ee59-4ae7-be96-690e96dc72a0.jpg"
            },
            {
                "object_name": "detectface/e029ef8c-ee59-4ae7-be96-690e96dc72a1.jpg",
                "url": "http://103.176.147.25:8004/q100-face-recogs/detectface/e029ef8c-ee59-4ae7-be96-690e96dc72a1.jpg"
            }
        ]
        }

        ```

        -   `bucket` (string): Tên của bucket MinIO nơi các ảnh khuôn mặt đã được tải lên.
        -   `urls` (list of strings): Danh sách các URL của các ảnh khuôn mặt đã được tải lên trong MinIO.
    -   **`500 Internal Server Error` (Thất bại):** Trả về một JSON chứa thông tin về lỗi.

        -   **Lỗi chuyển đổi URL sang Base64:**

            ```json
            {
              "detail": "Error converting URLs to base64: ..."
            }
            ```

        -   **Không có ảnh nào được chuyển đổi từ URL:**

            ```json
            {
              "detail": "No images converted from URLs"
            }
            ```

        -   **Không có khuôn mặt nào được phát hiện:**

            ```json
            {
              "detail": "No faces detected from the provided image URLs"
            }
            ```

        -   **Lỗi tải lên ảnh khuôn mặt đã cắt:**

            ```json
            {
              "detail": "Error uploading cropped face images: ..."
            }
            ```

-   **Xử lý:**

    1.  **Tạo `message_id` (nếu cần):** Nếu `message_id` không được cung cấp trong query parameters, một UUID mới sẽ được tạo.
    2.  **Chuyển đổi URL ảnh sang Base64:** Sử dụng adapter service (endpoint `/adapter/convert`) để chuyển đổi danh sách URL ảnh thành danh sách các chuỗi Base64.
    3.  **Phát hiện khuôn mặt:** Với mỗi ảnh (dạng Base64):
        -   Giải mã chuỗi Base64 thành ảnh NumPy.
        -   Sử dụng `retina_face_detector` để phát hiện khuôn mặt.
        -   Nếu khuôn mặt được phát hiện, cắt khuôn mặt và chuyển đổi thành Base64.
        -   Thêm chuỗi Base64 của khuôn mặt đã cắt vào danh sách `all_cropped_faces`.
    4.  **Tải ảnh khuôn mặt đã cắt lên MinIO:** Sử dụng adapter service (endpoint `/adapter/detectface/upload`) để tải danh sách các chuỗi Base64 (ảnh khuôn mặt đã cắt) lên MinIO.
    5.  **Trả về kết quả:** Trả về thông tin về bucket và danh sách các URL của các ảnh khuôn mặt đã tải lên từ adapter service.



### 3.3. Add Face API

-   **Tóm tắt:** Thêm hoặc cập nhật ảnh khuôn mặt cho một người.

-   **Mô tả:** Endpoint này nhận danh sách các URL ảnh và ID người dùng, xác thực các URL, trích xuất các embedding khuôn mặt và thêm chúng vào cơ sở dữ liệu.

-   **Phương thức:** `POST`

-   **Endpoint URL:** `/person/add-face/`

-   **Tham số:**

    -   **Request Body (JSON):**

        ```json
        {
          "selected_face_images": [
            "url_anh_1",
            "url_anh_2",
            ...
          ],
          "user_id": 12345
        }

        ```

        -   `selected_face_images` (list of strings, required): Danh sách các URL ảnh khuôn mặt đã được chọn để thêm hoặc cập nhật thông tin người dùng.
        -   `user_id` (integer, required): ID của người dùng.
-   **Request Body:**

    -   `selected_face_images`: Danh sách các URL ảnh khuôn mặt.
    -   `user_id`: ID của người dùng.
-   **Response:**

    -   **`200 OK` (Thành công):** Trả về một JSON cho biết ảnh đã được thêm thành công.

        ```json
        {
          "status": true,
          "content": 200,
          "user_id": 12345,
          "images": []
        }

        ```

        -   `status` (boolean): `true` cho biết thao tác thành công.
        -   `content` (integer): Mã trạng thái nội bộ (200).
        -   `user_id` (integer): ID của người dùng.
        -   `images` (list of strings): Danh sách các URL ảnh không được thêm vào (ví dụ: do đã tồn tại hoặc quá giống với các ảnh hiện có). Trong trường hợp thành công hoàn toàn (tất cả các ảnh đều được thêm), danh sách này sẽ trống.
    -   **`500 Internal Server Error` (Thất bại):** Trả về một JSON cho biết thao tác thất bại.

        ```json
        {
          "status": false,
          "content": 500,
          "user_id": 123465,
          "images": [
            "url_anh_1",
            "url_anh_2",
            ...
          ]
        }

        ```

        -   `status` (boolean): `false` cho biết thao tác thất bại.
        -   `content` (integer): Mã trạng thái nội bộ (500).
        -   `user_id` (integer, nullable): ID của người dùng (có thể là null trong một số trường hợp lỗi).
        -   `images` (list of strings): Danh sách các URL ảnh đã gây lỗi hoặc không được thêm vào.
-   **Xử lý:**

    1.  **Xác thực URL:** Đảm bảo rằng ít nhất một URL ảnh được cung cấp.
    2.  **Chuyển đổi URL sang Base64:** Sử dụng adapter service (endpoint `/adapter/convert`) để chuyển đổi danh sách URL ảnh thành danh sách các chuỗi Base64.
    3.  **Phát hiện và trích xuất khuôn mặt:** Sử dụng `retina_face_detector` để phát hiện khuôn mặt trong ảnh và `arc_face_recognizer` để trích xuất embedding khuôn mặt.
    4.  **Kiểm tra tính nhất quán hàng loạt:** Đảm bảo rằng tất cả các embedding thuộc về cùng một người.
    5.  **Xác định người dùng mới so với người dùng hiện tại:** Kiểm tra xem `user_id` đã tồn tại trong cơ sở dữ liệu hay chưa.
    6.  **Ngăn chặn tải lên "sai người":** So sánh embedding mới với các embedding hiện có trong cơ sở dữ liệu để ngăn chặn việc thêm ảnh của một người vào tài khoản của người khác.
    7.  **Phân loại URL để thêm so với không thêm:** Xác định những URL nào nên được thêm vào cơ sở dữ liệu và những URL nào nên bị bỏ qua (ví dụ: nếu chúng đã tồn tại hoặc quá giống với các ảnh hiện có).
    8.  **Thêm các embedding mới và lưu ảnh:** Thêm các embedding mới vào cơ sở dữ liệu và lưu các ảnh tương ứng vào thư mục của người dùng.

### 3.4. Infer Face Base64 API

- **Tóm tắt:** Suy luận và nhận dạng một khuôn mặt duy nhất từ một ảnh base64. 

- **Mô tả:** Endpoint này nhận một ảnh full-frame được mã hóa base64, phát hiện khuôn mặt, chọn khuôn mặt tốt nhất, căn chỉnh và cắt khuôn mặt, tính toán embedding khuôn mặt và tìm kiếm trong cơ sở dữ liệu khuôn mặt để nhận dạng người đó.

- **Phương thức:** `POST`

- **Endpoint URL:** `/infer/infer-face-base64`

- **Tham số:**
    - **Request Body (JSON):**

        ```json
        {
            "image": "chuỗi_base64_cua_anh"
        }
        ```

        - `image` (string, required): Ảnh khuôn mặt đã crop person dưới dạng chuỗi base64.

- **Request Body:**
    - `image`: Ảnh khuôn mặt đã crop person dưới dạng chuỗi base64.

- **Response:**
    - `200 - OK` **(Thành công):** Trả về một JSON chứa thông tin về kết quả suy luận.
        ```json
        { 
            "isKnown": true, 
            "user_id": 12345,
            "similarity": 0.85
        }
        ```
        - `isKnown` (boolean): `true` nếu khuôn mặt được nhận dạng trong cơ sở dữ liệu, `false` nếu không.
        - `user_id` (integer, nullable): ID của người dùng nếu khuôn mặt được nhận dạng, `null` nếu không.
        - `similarity` (float, nullable): Độ tương đồng giữa embedding khuôn mặt và embedding khuôn mặt phù hợp nhất trong cơ sở dữ liệu, `null` nếu không.
    - `400 - Bad Request` **(Lỗi):** Trả về một JSON chứa thông tin về lỗi.
        - **Ảnh base64 không hợp lệ:**
            ```json
            {
                "detail": "Invalid base64 image input"
            }
            ```
- **Xử lý:**
    1. **Giải mã base64:** Giải mã chuỗi base64 thành ảnh NumPy.
    2. **Phát hiện khuôn mặt:** Sử dụng `retina_face_detector` để phát hiện khuôn mặt và landmarks.
    3. **Chọn khuôn mặt tốt nhất:** Chọn khuôn mặt tốt nhất dựa trên kích thước, vị trí và khoảng cách giữa mắt.
    4. **Căn chỉnh và cắt khuôn mặt:** Căn chỉnh và cắt khuôn mặt bằng cách sử dụng landmarks đã phát hiện.
    5. **Tính toán embedding:** Tính toán embedding khuôn mặt bằng cách sử dụng arc_face_recognizer.
    6. **Tìm kiếm trong cơ sở dữ liệu:** Tìm kiếm trong cơ sở dữ liệu khuôn mặt bằng cách sử dụng `face_database.search_face` với `k=2` (tìm kiếm 2 kết quả phù hợp nhất).
    7. **Quyết định:** Xác định xem khuôn mặt có được nhận dạng hay không dựa trên độ tương đồng và ngưỡng (THRESHOLD và MARGIN).
    8. **Log kết quả:** Lưu ảnh gốc, ảnh đã căn chỉnh và thông tin suy luận vào thư mục log.
    9. **Trả về kết quả:** Trả về kết quả suy luận (isKnown, user_id, similarity).

### 3.5. Infer Face URL API

- **Tóm tắt:** Suy luận và nhận dạng một khuôn mặt duy nhất từ một URL ảnh.

- **Mô tả:** API này nhận một URL ảnh, tải ảnh về, phát hiện khuôn mặt, chọn khuôn mặt tốt nhất, căn chỉnh và cắt khuôn mặt, tính toán embedding khuôn mặt và tìm kiếm trong cơ sở dữ liệu khuôn mặt để nhận dạng người đó.

- **Phương thức:** `POST`

- **Endpoint:** `/infer/infer-face-url`

- **Tham số:**
    - **Request Body (JSON)**:
        ```json
        {
            "image_url": "url_cua_anh"
        }
        ```
        - `image_url` (string): Ảnh khuôn mặt đã crop person dưới dạng URL.

- **Request Body:**
    - `image_url`: Ảnh khuôn mặt đã crop person dưới dạng URL.

- **Response:**
    - `200 - OK` **(Thành công):** Trả về một JSON chứa thông tin về kết quả suy luận.
        ```json
        { 
            "isKnown": true, 
            "user_id": 12345,
            "similarity": 0.85
        }
        ```
        - `isKnown` (boolean): `true` nếu khuôn mặt được nhận dạng trong cơ sở dữ liệu, `false` nếu không.
        - `user_id` (integer, nullable): ID của người dùng nếu khuôn mặt được nhận dạng, `null` nếu không.
        - `similarity` (float, nullable): Độ tương đồng giữa embedding khuôn mặt và embedding khuôn mặt phù hợp nhất trong cơ sở dữ liệu, `null` nếu không.

    - `400 - Bad Request` **(Lỗi):** Trả về một JSON chứa thông tin về lỗi.
        - **Ảnh base64 không hợp lệ:**
            ```json
            {
                "detail": "Invalid base64 image data"
            }
            ```
    - `502 - Bad Gateway` **(Lỗi):** Trả về một JSON chứa thông tin về lỗi.
        - **Lỗi chuyển đổi URL sang base64:**
            ```json
            {
                "detail": "Error converting URL to base64: ..."
            }
            ```

- **Xử lý:**
    1. **Lấy base64 từ adapter:** Sử dụng adapter service (endpoint `/adapter/convert`) để chuyển đổi URL ảnh thành chuỗi base64.
    2. **Giải mã base64:** Giải mã chuỗi base64 thành ảnh NumPy.
    3. **Phát hiện khuôn mặt:** Sử dụng `retina_face_detector` để phát hiện khuôn mặt và landmarks.
    4. **Chọn khuôn mặt tốt nhất:** Chọn khuôn mặt tốt nhất dựa trên kích thước, vị trí và khoảng cách giữa mắt.
    5. **Căn chỉnh và cắ    ```json
            {
                "detail": "Error converting URL to base64: ..."
            }
            ```t khuôn mặt:** Căn chỉnh và cắt khuôn mặt bằng cách sử dụng landmarks đã phát hiện.
    6. **Tính toán embedding:** Tính toán embedding khuôn mặt bằng cách sử dụng `arc_face_recognizer`.
    7. **Tìm kiếm trong cơ sở dữ liệu:** Tìm kiếm trong cơ sở dữ liệu khuôn mặt bằng cách sử dụng `face_database.search_face` với `k=2` (tìm kiếm 2 kết quả phù hợp nhất).
    8. **Quyết định:** Xác định xem khuôn mặt có được nhận dạng hay không dựa trên độ tương đồng và ngưỡng (THRESHOLD và MARGIN).
    9. **Log kết quả:** Lưu ảnh gốc, ảnh đã căn chỉnh và thông tin suy luận vào thư mục log.
    10. **Trả về kết quả:** Trả về kết quả suy luận (isKnown, user_id, similarity).

### 3.6. Delete Person API

-   **Tóm tắt:** Xóa một người dùng khỏi cơ sở dữ liệu.

-   **Mô tả:** Endpoint này xóa một người dùng cụ thể khỏi cơ sở dữ liệu nhận dạng khuôn mặt. Nó loại bỏ thông tin người dùng khỏi FAISS index và metadata, đồng thời xóa thư mục chứa ảnh khuôn mặt của người đó.

-   **Phương thức:** `DELETE`

-   **Endpoint URL:** `/persons/{user_id}`

-   **Tham số:**

    -   `user_id` (path parameter, integer): ID của người dùng cần xóa.
-   **Request Body:** Không có.

-   **Response:**

    -   **`200 OK` (Thành công):** Trả về một JSON cho biết người dùng đã được xóa thành công.

        ```json
        {
          "status": true,
          "content": 200
        }

        ```

        -   `status` (boolean): `true` cho biết thao tác thành công.
        -   `content` (integer): Mã trạng thái nội bộ (200).
    -   **`500 Internal Server Error` (Thất bại):** Trả về một JSON cho biết người dùng không tồn tại.

        ```json
        {
          "status": true,
          "content": 500
        }

        ```

        -   `status` (boolean): `true` cho biết thao tác thất bại.
        -   `content` (integer): Mã trạng thái nội bộ (500).
-   **Xử lý:**

    1.  Kiểm tra xem người dùng có tồn tại trong metadata hay không.
    2.  Nếu người dùng tồn tại:
        -   Xóa người dùng khỏi `face_database` (FAISS index và metadata).
        -   Xóa thư mục chứa ảnh khuôn mặt của người dùng.
    3.  Trả về phản hồi thành công (200 OK) nếu xóa thành công.
    4.  Trả về phản hồi lỗi (500 Internal Server Error) nếu người dùng không tồn tại.

### 3.7. Delete Images API

-   **Tóm tắt:** Xóa ảnh khỏi hồ sơ của một người.

-   **Mô tả:** Endpoint này nhận một danh sách các URL ảnh và ID người dùng, chuyển đổi các URL ảnh thành embeddings và xóa các embeddings tương ứng khỏi cơ sở dữ liệu.

-   **Phương thức:** `POST`

-   **Endpoint URL:** `/person/delete-images/`

-   **Tham số:**

    -   **Request Body (JSON):**

        ```json
        {
          "message_id": "uuid_tin_nhan",
          "delete_face_images": [
            "url_anh_1",
            "url_anh_2",
            ...
          ],
          "user_id": 12345
        }

        ```

        -   `message_id` (string, optional): ID tin nhắn (UUID) tương ứng với thư mục tạm thời. Nếu để trống, server sẽ tự tạo.
        -   `delete_face_images` (list of strings, required): Danh sách các URL ảnh cần xóa.
        -   `user_id` (integer, required): ID của người dùng (kiểu số nguyên).
-   **Request Body:**

    -   `message_id`: ID tin nhắn (UUID) tương ứng với thư mục tạm thời.
    -   `delete_face_images`: Danh sách các URL ảnh cần xóa.
    -   `user_id`: ID của người dùng.
-   **Response:**

    -   **`200 OK` (Thành công):** Trả về một JSON cho biết ảnh đã được xóa thành công hoặc không tìm thấy ảnh phù hợp.

        ```json
        {
          "status": true,
          "detail": "Embeddings removed successfully"
        }

        ```

        Hoặc:

        ```json
        {
          "status": false,
          "detail": "No matching embeddings were found to remove"
        }

        ```

        -   `status` (boolean): `true` nếu embeddings đã được xóa thành công, `false` nếu không tìm thấy embeddings phù hợp.
        -   `detail` (string): Một thông báo mô tả kết quả.
    -   **`500 Internal Server Error` (Lỗi):** Trả về một JSON chứa thông tin về lỗi.

        -   **Không cung cấp URL ảnh:**

            ```json
            {
                "detail": "Please provide image URLs."
            }
            ```
        -   **`delete_face_images` là danh sách trống:**

            ```json
            {
              "detail": "delete_face_images must be a non-empty list"
            }

            ```

        -   **Lỗi chuyển đổi URL sang base64:**

            ```json
            {
              "detail": "Error converting image URLs to base64 via adapter: ..."
            }

            ```

        -   **Không có ảnh base64 nào được lấy từ URL đã cung cấp:**

            ```json
            {
              "detail": "No base64 images obtained from the provided URLs."
            }

            ```

        -   **Không thể tính toán embeddings cho ảnh đã cung cấp:**

            ```json
            {
              "detail": "Could not compute embeddings for provided images."
            }

            ```

-   **Xử lý:**

    1.  **Xác thực đầu vào:** Kiểm tra xem `delete_face_images` có được cung cấp và không trống hay không.
    2.  **Tạo `message_id` (nếu cần):** Nếu `message_id` không được cung cấp, một UUID mới sẽ được tạo.
    3.  **Chuyển đổi URL ảnh sang Base64:** Sử dụng adapter service (endpoint `/adapter/convert`) để chuyển đổi danh sách URL ảnh thành danh sách các chuỗi Base64.
    4.  **Tính toán embeddings:** Với mỗi ảnh (dạng Base64):
        -   Giải mã chuỗi Base64 thành ảnh NumPy.
        -   Sử dụng `arc_face_recognizer` để tính toán embedding.
    5.  **Xóa embeddings:** Sử dụng `face_database.remove_selected_embeddings` để xóa các embeddings tương ứng với `user_id`.
    6.  **Trả về kết quả:** Trả về một JSON cho biết thao tác đã thành công hay không.
