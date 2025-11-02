from flask import Flask, request, send_file, jsonify
import subprocess
import tempfile
import os

app = Flask(__name__)

# ===============================
# 1️⃣ Allow React requests (CORS)
# ===============================
from flask_cors import CORS
CORS(app)  # Allows all origins by default

# ===============================
# 2️⃣ Download route
# ===============================
@app.route("/download", methods=["POST", "OPTIONS"])
def download_video():
    # Handle CORS preflight
    if request.method == "OPTIONS":
        return ("", 200)

    # ===============================
    # 3️⃣ Get video URL
    # ===============================
    data = request.get_json()
    if not data or "url" not in data:
        return jsonify({"error": "No URL provided"}), 400

    url = data["url"]
    temp_dir = tempfile.gettempdir()
    output_path = os.path.join(temp_dir, f"video_{int(os.times()[4])}.mp4")

    # ===============================
    # 4️⃣ Run yt-dlp command
    # ===============================
    command = [
        "yt-dlp",
        "-f", "bv*+ba/b",
        "-o", output_path,
        url
    ]

    try:
        subprocess.run(command, check=True, capture_output=True)
    except subprocess.CalledProcessError as e:
        return jsonify({
            "error": "Download failed",
            "details": e.stderr.decode("utf-8", errors="ignore")
        }), 500

    # ===============================
    # 5️⃣ Send the file back
    # ===============================
    if not os.path.exists(output_path):
        return jsonify({"error": "Output file not found"}), 500

    response = send_file(
        output_path,
        as_attachment=True,
        download_name="video.mp4",
        mimetype="video/mp4"
    )

    # Cleanup after sending
    @response.call_on_close
    def cleanup():
        try:
            os.remove(output_path)
        except Exception:
            pass

    return response


# ===============================
# 6️⃣ Run server (for local or Render)
# ===============================
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=10000)
