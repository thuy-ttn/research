# Khai thác lỗ hổng giải nén tệp tin trên ứng dụng Python để tấn công thực thi mã

Một trong những phương thức có thể dẫn đến tấn công thực thi mã (code execution) trong PHP là khai thác lỗ hổng trong logic lưu tệp tin được upload lên server một cách kém an toàn. Nếu có thể upload file PHP bất kì bằng cách đánh lừa logic của ứng dụng, ta có thể thực thi mã PHP tùy ý.

Tuy nhiên đối với các web frameworks mới được viết bằng Go, Python hay Node.js, câu chuyện lại hoàn toàn khác. Kể cả khi có thể upload các file với định dạng .py hoặc .js lên server, việc truy cập vào các đối tượng này bằng URL thường đi vào ngõ cụt bởi ứng dụng không set các routes đến các file này (ứng dụng không mapping các file source code ta upload lên với một URL trong ứng dụng). Kể cả khi ta có thể truy cập trực tiếp vào các file này bằng URL, ta cũng không thể tấn công thực thi mã, do ứng dụng xử lý các file này dưới dạng file tĩnh, và chỉ trả về một file source code dạng plain text.


Bài viết hôm nay sẽ giải thích phương pháp tấn công thực thi mã trong các trường hợp kể trên, với ví dụ là một ứng dụng Python với chức năng upload file nén lên server.
## Phát hiện lỗ hổng
Ứng dụng Python cho phép người dùng upload một file nén lên server. Dưới đây là phần code của chức năng upload.         
```python
def upload_zip():
    uploaded_file = request.files['file']
    filename = uploaded_file.filename
    file_ext = filename.split(".")[1]
    
    if not zipfile.is_zipfile(uploaded_file.stream) or file_ext not in app.config['UPLOAD_EXTENSIONS']:
         abort(400)
    
    file_path = os.path.join(app.config['UPLOADED_PATH'],filename)
    uploaded_file.save(file_path)
    #extract file
    extract_path = os.path.join(app.config['UPLOADED_PATH'], filename[:-4])
    files = unzip_file(file_path, extract_path)

    return files
```
Chức năng upload của server sẽ nhận file zip từ phía người dùng, tiến hành kiểm tra file vừa được upload, để đảm bảo người dùng upload file hợp lệ, sau đó lưu file ở phía server.
Hàm `unzip_file` được gọi với 2 biến truyền vào đó là path của file zip và địa chỉ để giải nén file.
```python
def unzip_file(path_zip_file, destination):
    try:
        if not os.path.exists(destination):
            try:
                os.makedirs(os.path.dirname(destination))
            except OSError as exc:
                if exc.errno != errno.EEXIST:
                    print("Race condition!!!")

        #extract file
        files = subprocess.Popen([f'unzip -o {path_zip_file} -d {destination}'], stdout=subprocess.PIPE, shell=True, text=True).stdout.read()
        return files
    except Exception as e:
        return "[ERROR] Unzipping error: " + str(e)

```
Hàm `unzip_file` trả về output của câu lệnh unzip được thực thi phía server. Ứng dụng thực hiện giải nén file zip và lưu tại thư mục `{webroot}/uploads/{tên_file_zip}`.
Nếu nhìn vào dòng code sau
```python
extract_path = os.path.join(app.config['UPLOADED_PATH'], filename[:-4])
```
ta thấy biến `filename` được kiểm soát bởi người dùng. Nếu ta set giá trị của `filename` thành `../../test.zip` ta có thể khai thác được lỗ hổng Path Traversal.
Lỗ hổng này cho phép ta giải nén file vào vị trí bất kì trên server thay vì thư mục gốc ban đầu.  
![Image 1](/Article_1/img_1.png)

## Phương thức tấn công
Sau khi phát hiện lỗ hổng Path Traversal, ta cần tìm cách khai thác lỗ hổng đó. 
Ứng dụng đang sử dụng Flask framework được viết bằng Python, ta có thể ghi đè file `__init__.py` để thực hiện tấn công thực thi mã.
Dựa theo [tài liệu](https://docs.python.org/2.7/tutorial/modules.html#packages) của Python về file `__init__.py` 

>The __init__.py files are required to make Python treat the directories as containing packages; this is done to prevent directories with a common name, such as string, from unintentionally hiding valid modules that occur later on the module search path. In the simplest case, __init__.py can just be an empty file, but it can also execute initialization code for the package or set the __all__ variable, described later.
File `__init__.py` là một file bắt buộc để Python hiểu rằng đó là một package.
Trong ứng dụng Flask, thường các config hoặc settings của ứng dụng sẽ được lưu thành các package riêng, tức là có sử dụng file `__init__.py`.
Nếu ta sửa được các file `__init__.py` này, ta có thể tấn công thực thi mã.

Tuy nhiên, điều kiện để chạy được file `__init__.py` mà ta vừa tạo, đó là server cần được restart.
Ở ví dụ này, ta đang chạy Flask server với `debug = True`, tức là mỗi lần file Python được sửa, server sẽ tự động restart.

## Payload thực thi
Trong ví dụ này, ứng dụng có một directory là `config`, bao gồm 1 file `__init__.py` và 1 file `settings.py`.
File `app.py` là file main của ứng dụng, import `settings.py` từ directory `config`,
tức là nếu ta ghi đè được mã vào file `config/__init__.py`, ta có thể tấn công thực thi mã.  
Để tấn công lỗ hổng này, ta sẽ tạo 1 file zip chứa file `__init__.py`, chứa mã tạo reverse shell đến máy attacker.
Tên file zip sẽ lợi dụng lỗ hổng Path Traversal, để sau khi giải nén, ta được file `__init__.py` trong thư mục `config`  
![Image_2](/Article_1/img_2.png)  
Sau khi upload file lên server thành công, ứng dụng sẽ tự động reload và chạy file `config/__init__.py`, trả về máy attacker một reverse shell.  
![Image_3](/Article_1//img_3.png)

## Tính khả dụng & Cách phòng chống
Việc khai thác lỗ hổng này phụ thuộc vào nhiều yếu tố:
- Server có thực hiện restart: Trên thực tế, việc server chạy debug mode như trong ví dụ rất hiếm gặp. Mặc dù thế, ta vẫn có thể khai thác lỗ hổng bằng cách chờ server được restart hoặc reload code.
- File `__init__.py` được sử dụng trong server: Đối với nhiều ứng dụng, việc tìm một file `__init__.py` có thể không dễ như trong directory `config`. Tuy nhiên ở mức độ ứng dụng web, các package thường được sử dụng có thể kể đến như `model`, `utils`, `view`.
Ngoài ra còn có các package directory dạng framework thường được sử dụng ví dụ như Flask, Django. 
Nếu ứng dụng được chạy trên Ubuntu Linux, ta có thể đoán được địa chỉ của Python pip được cài đặt, ở `/home/<user>/.local/lib/python3.8/site-packages/pip` hoặc ở `/venv/lib/python3.8/site-packages/pip/__init__.py`.

Để phòng chống lỗ hổng này, lập trình viên nên xử lý input đầu vào từ phía người dùng. Hàm `werkzeug.secure_filename` được tích hợp sẵn trong Flask hỗ trợ làm sạch input đầu vào đối với filename.   
