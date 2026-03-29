import discord
import os
import yt_dlp
import asyncio

# ---------------------------------------------------------
# CẤU HÌNH BOT (THAY THÔNG TIN CỦA BẠN VÀO ĐÂY)
# ---------------------------------------------------------
TOKEN = 'MTQ4NzgyNzY5MjExMDg3Njc0Mg.GPsoFF.k3C0zPLbhUstXs9aRqCyPFp2DyGqhpYZhdudCc'
VOICE_CHANNEL_ID = 891701528216350768 # <--- DÁN DÃY SỐ ID KÊNH THOẠI VÀO ĐÂY (Không cần dấu ngoặc kép)

# Bật Intent
intents = discord.Intents.default()
intents.message_content = True
client = discord.Client(intents=intents)

# ---------------------------------------------------------
# HÀM TẢI VIDEO BẰNG YT-DLP
# ---------------------------------------------------------
def download_video(link, filename):
    ydl_opts = {
        'outtmpl': filename,
        'format': 'bestvideo+bestaudio/best',
        'noplaylist': True,
        'quiet': True,
        'no_warnings': True
    }
    with yt_dlp.YoutubeDL(ydl_opts) as ydl:
        ydl.download([link])

# ---------------------------------------------------------
# SỰ KIỆN KHI BOT BẬT LÊN
# ---------------------------------------------------------
@client.event
async def on_ready():
    print(f'✅ Bot đã online: {client.user}')
    
    # --- LOGIC TỰ ĐỘNG VÀO PHÒNG THOẠI ---
    try:
        channel = client.get_channel(VOICE_CHANNEL_ID)
        if channel and isinstance(channel, discord.VoiceChannel):
            # Kết nối vào phòng
            await channel.connect()
            # Đổi trạng thái: Tắt mic (self_mute) và Tắt loa (self_deaf)
            await channel.guild.change_voice_state(channel=channel, self_mute=True, self_deaf=True)
            print(f'🎧 Đã chui vào phòng thoại: [{channel.name}] (Đang tắt mic/loa 24/24)')
        else:
            print('⚠️ CẢNH BÁO: Không tìm thấy phòng thoại! Bạn đã dán đúng ID chưa?')
    except Exception as e:
        print(f'❌ Lỗi khi vào phòng thoại: Bạn đã cài PyNaCl chưa? (Chi tiết lỗi: {e})')
    
    print('Đang chờ mọi người gửi link TikTok...')
    print('-' * 40)

# ---------------------------------------------------------
# SỰ KIỆN KHI CÓ TIN NHẮN MỚI
# ---------------------------------------------------------
@client.event
async def on_message(message):
    if message.author == client.user:
        return

    # Kiểm tra xem tin nhắn có chứa "tiktok.com" không
    if "tiktok.com" in message.content.lower():
        words = message.content.split()
        link = ""
        for word in words:
            if "tiktok.com" in word.lower():
                link = word
                break
        
        if not link:
            return

        # Xóa tin nhắn gốc của user
        try:
            await message.delete()
            print(f"[LOG] Đã xóa tin nhắn chứa link của {message.author.name}")
        except Exception:
            pass # Bỏ qua lỗi nếu bot chưa được cấp quyền xóa tin nhắn

        # Thông báo đang tải
        noti_msg = await message.channel.send(f"⏳ Đang tải video TikTok cho {message.author.mention}, đợi xíu nhé...")
        filename = f"tiktok_{message.id}.mp4"
        
        try:
            await asyncio.to_thread(download_video, link, filename)
            
            if os.path.exists(filename):
                file_size = os.path.getsize(filename) / (1024 * 1024)
                
                if file_size > 25:
                    await noti_msg.edit(content=f"❌ Video quá nặng ({file_size:.2f}MB). Discord free chỉ cho gửi tối đa 25MB thôi.")
                else:
                    with open(filename, 'rb') as f:
                        await message.channel.send(f"✅ Video của {message.author.mention} đây:", file=discord.File(f, "tiktok.mp4"))
                    await noti_msg.delete()
                
                os.remove(filename)
                print(f"[SUCCESS] Đã xử lý xong video cho {message.author.name}")
                
            else:
                await noti_msg.edit(content=f"❌ {message.author.mention} Không tải được video.")
                
        except Exception as e:
            print(f"[LỖI] Chi tiết lỗi: {e}")
            await noti_msg.edit(content=f"❌ {message.author.mention} Đã có lỗi hệ thống xảy ra!")
            if os.path.exists(filename):
                os.remove(filename)

# Khởi chạy bot
client.run('MTQ4NzgyNzY5MjExMDg3Njc0Mg.GPsoFF.k3C0zPLbhUstXs9aRqCyPFp2DyGqhpYZhdudCc')
