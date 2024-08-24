from flask import Flask, render_template, request, redirect, url_for, flash, send_from_directory
import yt_dlp
import os
import shutil
import subprocess

app = Flask(__name__)
app.secret_key = 'your_secret_key'

def cleanup_static_folder():
    # Xóa tất cả các tệp trong thư mục static trước khi quay lại trang chính
    folder = 'static'
    for filename in os.listdir(folder):
        file_path = os.path.join(folder, filename)
        try:
            if os.path.isfile(file_path) or os.path.islink(file_path):
                os.unlink(file_path)
        except Exception as e:
            print(f'Failed to delete {file_path}. Reason: {e}')

@app.route('/')
def index():
    cleanup_static_folder()  # Xóa các tệp trước khi tải trang index
    return render_template('index.html')

@app.route('/get_video_info', methods=['POST'])
def get_video_info():
    url = request.form['url']
    
    ydl_opts = {
        'format': 'bestvideo+bestaudio/best',  # Tải cả video và âm thanh tốt nhất
        'quiet': True,
    }
    
    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info_dict = ydl.extract_info(url, download=False)
            formats = info_dict.get('formats', [])
        
        if not formats:
            flash("Không tìm thấy video hoặc URL không hợp lệ.")
            return redirect(url_for('index'))

        # In ra tất cả các định dạng để kiểm tra
        for f in formats:
            print(f"Format ID: {f.get('format_id')}, Height: {f.get('height')}, Ext: {f.get('ext')}, Format Note: {f.get('format_note')}")

        # Lưu tất cả các định dạng mà không cần lọc theo chiều cao
        unique_formats = {}
        for f in formats:
            height = f.get("height")
            if height:  # Lưu tất cả các định dạng có chiều cao
                unique_formats[height] = {
                    "format_id": f.get("format_id"),
                    "ext": f.get("ext"),
                    "height": f.get("height"),
                    "quality": f.get("format_note"),
                    "filesize": f.get("filesize"),
                    "url": f.get("url"),
                }

        video_formats = list(unique_formats.values())

        return render_template('video_options.html', video_formats=video_formats, url=url)
    except Exception as e:
        flash(f"Đã xảy ra lỗi: {e}")
        return redirect(url_for('index'))

@app.route('/download_video/<format_id>', methods=['GET'])
def download_video(format_id):
    url = request.args.get('url')
    
    video_opts = {
        'format': f'{format_id}+bestaudio',  # Tải cả video và âm thanh
        'outtmpl': 'downloaded_video.mp4',
    }
    
    try:
        # Tải video và âm thanh cùng lúc
        with yt_dlp.YoutubeDL(video_opts) as ydl:
            ydl.download([url])
        
        video_filename = 'downloaded_video.mp4'
        
        # Di chuyển tệp vào thư mục static để tải về
        if not os.path.exists('static'):
            os.makedirs('static')
        shutil.move(video_filename, os.path.join('static', video_filename))

        return redirect(url_for('success', filename=video_filename))
    except Exception as e:
        flash(f"Đã xảy ra lỗi: {e}")
        return redirect(url_for('error'))

@app.route('/render_video/<format_id>', methods=['GET'])
def render_video(format_id):
    url = request.args.get('url')
    
    # Tải video và audio riêng biệt cho độ phân giải trên 1280p
    video_opts = {
        'format': f'{format_id}',  # Tải video riêng biệt
        'outtmpl': 'rendered_video.mp4',
    }

    audio_opts = {
        'format': 'bestaudio',  # Tải âm thanh riêng biệt
        'outtmpl': 'rendered_audio.m4a',
    }
    
    try:
        # Tải video
        with yt_dlp.YoutubeDL(video_opts) as ydl:
            ydl.download([url])
        
        # Tải âm thanh
        with yt_dlp.YoutubeDL(audio_opts) as ydl:
            ydl.download([url])
        
        # Sử dụng FFmpeg để kết hợp video và âm thanh
        video_filename = 'rendered_video.mp4'
        audio_filename = 'rendered_audio.m4a'  # Giả định âm thanh được tải dưới định dạng m4a
        output_filename = 'final_video.mp4'
        
        # Kiểm tra xem các tệp có tồn tại không trước khi thực hiện lệnh ffmpeg
        if os.path.exists(video_filename) and os.path.exists(audio_filename):
            command = f'ffmpeg -i {video_filename} -i {audio_filename} -c copy {output_filename}'
            subprocess.run(command, shell=True, check=True)
            
            # Di chuyển tệp vào thư mục static để tải về
            if not os.path.exists('static'):
                os.makedirs('static')
            shutil.move(output_filename, os.path.join('static', output_filename))

            # Xóa các tệp tạm sau khi ghép thành công
            os.remove(video_filename)
            os.remove(audio_filename)
    
            return redirect(url_for('success', filename=output_filename))
        else:
            flash("Tệp video hoặc âm thanh không tồn tại.")
            return redirect(url_for('error'))
    except Exception as e:
        flash(f"Đã xảy ra lỗi: {e}")
        return redirect(url_for('error'))

@app.route('/success')
def success():
    filename = request.args.get('filename')
    return render_template('success.html', filename=filename)

@app.route('/download_file/<filename>')
def download_file(filename):
    return send_from_directory(directory='static', path=filename, as_attachment=True)

@app.route('/error')
def error():
    return render_template('error.html')

if __name__ == '__main__':
    app.run(debug=True)
