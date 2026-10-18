package com.xiaomi.transfer

import android.Manifest
import android.app.Activity
import android.content.Intent
import android.content.pm.PackageManager
import android.graphics.Bitmap
import android.graphics.Color
import android.graphics.Typeface
import android.graphics.drawable.GradientDrawable
import android.media.MediaScannerConnection
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.os.Environment
import android.provider.Settings
import android.view.Gravity
import android.view.View
import android.view.WindowManager
import android.widget.*
import org.json.JSONArray
import org.json.JSONObject
import java.io.*
import java.net.*
import java.text.SimpleDateFormat
import java.util.*
import java.util.concurrent.Executors
import java.util.zip.Deflater
import java.util.zip.ZipEntry
import java.util.zip.ZipOutputStream

class MainActivity : Activity() {

    private val PORT = 8080
    private val SAVE_DIR = File("/storage/emulated/0/DCIM/iPhone_Transfer")
    private val FOLDER_MAP = mapOf(
        "transfer" to listOf("/storage/emulated/0/DCIM/iPhone_Transfer"),
        "camera" to listOf("/storage/emulated/0/DCIM/Camera"),
        "screenshots" to listOf(
            "/storage/emulated/0/DCIM/Screenshots",
            "/storage/emulated/0/Pictures/Screenshots"
        ),
        "download" to listOf("/storage/emulated/0/Download")
    )
    private val ALLOWED_EXTS = setOf(
        ".jpg", ".jpeg", ".png", ".gif", ".webp", ".heic", ".dng",
        ".mp4", ".mov", ".m4v", ".mkv", ".avi"
    )
    private val VIDEO_EXTS = setOf(".mp4", ".mov", ".m4v", ".mkv", ".avi")
    private val MIME_MAP = mapOf(
        ".jpg" to "image/jpeg", ".jpeg" to "image/jpeg", ".png" to "image/png",
        ".gif" to "image/gif", ".webp" to "image/webp", ".heic" to "image/heic",
        ".dng" to "image/x-adobe-dng", ".mp4" to "video/mp4", ".mov" to "video/quicktime",
        ".m4v" to "video/x-m4v", ".mkv" to "video/x-matroska", ".avi" to "video/x-msvideo"
    )

    private var serverSocket: ServerSocket? = null
    private val threadPool = Executors.newCachedThreadPool()
    private lateinit var statusTv: TextView
    private lateinit var urlTv: TextView
    private lateinit var qrImageView: ImageView
    private lateinit var permBtn: Button
    private lateinit var logTv: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 保持小米手機螢幕常亮不休眠
        window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)

        buildNativeUI()
        checkAndUpdatePermissionUI()
        startHttpServer()
        refreshNetworkAndQR()
    }

    override fun onResume() {
        super.onResume()
        checkAndUpdatePermissionUI()
        refreshNetworkAndQR()
    }

    override fun onDestroy() {
        super.onDestroy()
        try { serverSocket?.close() } catch (_: Exception) {}
    }

    private fun buildNativeUI() {
        val rootScroll = ScrollView(this).apply {
            setBackgroundColor(Color.parseColor("#F2F2F7"))
            isFillViewport = true
        }
        val container = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            gravity = Gravity.CENTER_HORIZONTAL
            setPadding(48, 64, 48, 64)
        }

        val card = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            gravity = Gravity.CENTER_HORIZONTAL
            setPadding(56, 56, 56, 56)
            background = GradientDrawable().apply {
                setColor(Color.WHITE)
                cornerRadius = 48f
            }
        }

        val titleTv = TextView(this).apply {
            text = "📱 小米 ⇄ iPhone 雙向互傳"
            textSize = 22f
            setTypeface(null, Typeface.BOLD)
            setTextColor(Color.parseColor("#1C1C1E"))
            gravity = Gravity.CENTER
        }
        card.addView(titleTv)

        val subTv = TextView(this).apply {
            text = "⚡ 獨立原生 App・保留原始拍攝時間與 EXIF"
            textSize = 13f
            setTextColor(Color.parseColor("#1E7E34"))
            setPadding(0, 12, 0, 28)
            gravity = Gravity.CENTER
        }
        card.addView(subTv)

        permBtn = Button(this).apply {
            text = "🔑 點此授予「小米相簿讀寫權限」"
            textSize = 14f
            setTextColor(Color.WHITE)
            background = GradientDrawable().apply {
                setColor(Color.parseColor("#FF9500"))
                cornerRadius = 28f
            }
            setOnClickListener { requestStoragePermission() }
        }
        val permParams = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT
        ).apply { bottomMargin = 28 }
        card.addView(permBtn, permParams)

        statusTv = TextView(this).apply {
            textSize = 14f
            setTextColor(Color.parseColor("#3A3A3C"))
            gravity = Gravity.CENTER
            setPadding(16, 8, 16, 16)
        }
        card.addView(statusTv)

        qrImageView = ImageView(this).apply {
            scaleType = ImageView.ScaleType.FIT_CENTER
        }
        val qrSize = (resources.displayMetrics.widthPixels * 0.62).toInt().coerceAtMost(680)
        val qrParams = LinearLayout.LayoutParams(qrSize, qrSize).apply {
            topMargin = 12
            bottomMargin = 16
        }
        card.addView(qrImageView, qrParams)

        urlTv = TextView(this).apply {
            textSize = 16f
            setTypeface(Typeface.MONOSPACE, Typeface.BOLD)
            setTextColor(Color.parseColor("#007AFF"))
            gravity = Gravity.CENTER
            setPadding(0, 8, 0, 28)
        }
        card.addView(urlTv)

        val refreshBtn = Button(this).apply {
            text = "🔄 已開個人熱點？點此刷新條碼"
            textSize = 15f
            setTextColor(Color.WHITE)
            background = GradientDrawable().apply {
                setColor(Color.parseColor("#007AFF"))
                cornerRadius = 28f
            }
            setOnClickListener { refreshNetworkAndQR() }
        }
        val btnParams = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT
        )
        card.addView(refreshBtn, btnParams)

        logTv = TextView(this).apply {
            text = "📋 傳輸紀錄：\n等待 iPhone 掃碼連線中..."
            textSize = 12f
            setTextColor(Color.parseColor("#636366"))
            setPadding(24, 36, 24, 12)
            gravity = Gravity.START
        }
        card.addView(logTv, LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT
        ))

        container.addView(card, LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT
        ))
        rootScroll.addView(container)
        setContentView(rootScroll)
    }

    private fun checkAndUpdatePermissionUI() {
        val hasPerm = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            Environment.isExternalStorageManager()
        } else {
            checkSelfPermission(Manifest.permission.WRITE_EXTERNAL_STORAGE) == PackageManager.PERMISSION_GRANTED
        }
        if (hasPerm) {
            permBtn.visibility = View.GONE
            try { SAVE_DIR.mkdirs() } catch (_: Exception) {}
        } else {
            permBtn.visibility = View.VISIBLE
        }
    }

    private fun requestStoragePermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            try {
                val intent = Intent(
                    Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION,
                    Uri.parse("package:$packageName")
                )
                startActivity(intent)
            } catch (_: Exception) {
                val intent = Intent(Settings.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION)
                startActivity(intent)
            }
        } else {
            requestPermissions(
                arrayOf(Manifest.permission.READ_EXTERNAL_STORAGE, Manifest.permission.WRITE_EXTERNAL_STORAGE),
                1001
            )
        }
    }

    private fun appendLog(msg: String) {
        val timeStr = SimpleDateFormat("HH:mm:ss", Locale.getDefault()).format(Date())
        runOnUiThread {
            val lines = logTv.text.toString().split("\n").takeLast(8)
            logTv.text = (lines + "[$timeStr] $msg").joinToString("\n")
        }
    }

    private fun refreshNetworkAndQR() {
        threadPool.execute {
            val (bestIp, isLanReady) = getBestLanIp()
            val url = "http://$bestIp:$PORT"
            val qrBmp = generateQrBitmap(url)
            runOnUiThread {
                urlTv.text = url
                if (qrBmp != null) qrImageView.setImageBitmap(qrBmp)
                if (isLanReady) {
                    statusTv.text = "✅ 熱點/區網已就緒！請拿 iPhone 相機掃描下方條碼："
                    statusTv.setTextColor(Color.parseColor("#1C1C1E"))
                } else {
                    statusTv.text = "⚠️ 目前僅偵測到行動網路 ($bestIp)\n請先開啟小米「個人熱點」或連上 Wi-Fi，再點擊下方刷新按鈕！"
                    statusTv.setTextColor(Color.parseColor("#D9534F"))
                }
            }
        }
    }

    private fun getBestLanIp(): Pair<String, Boolean> {
        val ips = mutableSetOf<String>()
        try {
            val interfaces = NetworkInterface.getNetworkInterfaces()
            while (interfaces.hasMoreElements()) {
                val nif = interfaces.nextElement()
                if (!nif.isUp || nif.isLoopback) continue
                val addrs = nif.inetAddresses
                while (addrs.hasMoreElements()) {
                    val addr = addrs.nextElement()
                    if (addr is Inet4Address) {
                        val ip = addr.hostAddress ?: continue
                        if (!ip.startsWith("127.") && !ip.startsWith("100.")) {
                            ips.add(ip)
                        }
                    }
                }
            }
        } catch (_: Exception) {}

        for (target in listOf("192.168.43.1", "192.168.1.1", "172.20.10.1")) {
            var s: DatagramSocket? = null
            try {
                s = DatagramSocket()
                s.connect(InetAddress.getByName(target), 80)
                val ip = s.localAddress?.hostAddress ?: ""
                if (ip.isNotEmpty() && !ip.startsWith("127.")) ips.add(ip)
            } catch (_: Exception) {
            } finally {
                s?.close()
            }
        }

        val lanIps = ips.filter { it.startsWith("192.168.") || it.startsWith("172.") }.sorted()
        if (lanIps.isNotEmpty()) return Pair(lanIps.first(), true)
        val otherIps = ips.sorted()
        return Pair(otherIps.firstOrNull() ?: "192.168.43.1", false)
    }

    /** 純 Kotlin 零套件產生 Version 2-L QR Code 高清 Bitmap */
    private fun generateQrBitmap(text: String): Bitmap? {
        val raw = text.toByteArray(Charsets.US_ASCII)
        if (raw.size > 32) return null

        val sb = StringBuilder("0100")
        sb.append(raw.size.toString(2).padStart(8, '0'))
        for (b in raw) {
            sb.append((b.toInt() and 0xFF).toString(2).padStart(8, '0'))
        }
        sb.append("0000")
        val bits = sb.toString()

        val dataCw = mutableListOf<Int>()
        var idx = 0
        while (idx < bits.length) {
            dataCw.add(bits.substring(idx, idx + 8).toInt(2))
            idx += 8
        }
        val pad = intArrayOf(0xEC, 0x11)
        while (dataCw.size < 34) {
            dataCw.add(pad[(dataCw.size - (raw.size + 2)) % 2])
        }

        val gfExp = IntArray(512) { 1 }
        val gfLog = IntArray(256)
        var v = 1
        for (i in 0 until 255) {
            gfExp[i] = v
            gfExp[i + 255] = v
            gfLog[v] = i
            v = (v shl 1) xor (if ((v and 0x80) != 0) 0x11D else 0)
        }
        var gen = intArrayOf(1)
        for (i in 0 until 10) {
            val nxt = IntArray(gen.size + 1)
            for (j in gen.indices) {
                val c = gen[j]
                nxt[j] = nxt[j] xor c
                if (c != 0) nxt[j + 1] = nxt[j + 1] xor gfExp[gfLog[c] + i]
            }
            gen = nxt
        }
        val msg = IntArray(44)
        for (i in 0 until 34) msg[i] = dataCw[i]
        for (i in 0 until 34) {
            val coef = msg[i]
            if (coef != 0) {
                for (j in 1..10) {
                    if (gen[j] != 0) msg[i + j] = msg[i + j] xor gfExp[gfLog[coef] + gfLog[gen[j]]]
                }
            }
        }
        val bitStream = StringBuilder()
        for (b in dataCw) bitStream.append(b.toString(2).padStart(8, '0'))
        for (i in 34 until 44) bitStream.append(msg[i].toString(2).padStart(8, '0'))
        bitStream.append("0000000")

        val N = 25
        val grid = Array(N) { arrayOfNulls<Int>(N) }
        for ((r0, c0) in listOf(0 to 0, 0 to 18, 18 to 0)) {
            for (dr in -1..7) {
                for (dc in -1..7) {
                    if (r0 + dr in 0 until N && c0 + dc in 0 until N) grid[r0 + dr][c0 + dc] = 0
                }
            }
            for (dr in 0..6) {
                for (dc in 0..6) {
                    grid[r0 + dr][c0 + dc] = if (dr == 0 || dr == 6 || dc == 0 || dc == 6 || (dr in 2..4 && dc in 2..4)) 1 else 0
                }
            }
        }
        for (dr in -2..2) {
            for (dc in -2..2) {
                grid[18 + dr][18 + dc] = if (kotlin.math.abs(dr) == 2 || kotlin.math.abs(dc) == 2 || (dr == 0 && dc == 0)) 1 else 0
            }
        }
        for (i in 8..16) {
            val bit = if (i % 2 == 0) 1 else 0
            grid[6][i] = bit
            grid[i][6] = bit
        }
        grid[17][8] = 1
        for (i in (0..8) + (17..24)) {
            if (grid[8][i] == null) grid[8][i] = 0
            if (grid[i][8] == null) grid[i][8] = 0
        }

        var bIdx = 0
        var upward = true
        var col = 24
        while (col > 0) {
            if (col == 6) col -= 1
            val rows = if (upward) (24 downTo 0) else (0..24)
            for (r in rows) {
                for (c in listOf(col, col - 1)) {
                    if (grid[r][c] == null) {
                        val b = bitStream[bIdx] - '0'
                        bIdx++
                        grid[r][c] = b xor (if ((r + c) % 2 == 0) 1 else 0)
                    }
                }
            }
            upward = !upward
            col -= 2
        }

        val fmt = 0x77C4
        val c1 = listOf(8 to 0, 8 to 1, 8 to 2, 8 to 3, 8 to 4, 8 to 5, 8 to 7, 8 to 8, 7 to 8, 5 to 8, 4 to 8, 3 to 8, 2 to 8, 1 to 8, 0 to 8)
        val c2 = listOf(24 to 8, 23 to 8, 22 to 8, 21 to 8, 20 to 8, 19 to 8, 18 to 8, 8 to 17, 8 to 18, 8 to 19, 8 to 20, 8 to 21, 8 to 22, 8 to 23, 8 to 24)
        for (i in 0 until 15) {
            val bit = (fmt shr (14 - i)) and 1
            grid[c1[i].first][c1[i].second] = bit
            grid[c2[i].first][c2[i].second] = bit
        }

        val padQ = 3
        val M = N + padQ * 2
        val scale = 20
        val bmp = Bitmap.createBitmap(M * scale, M * scale, Bitmap.Config.ARGB_8888)
        for (r in 0 until M) {
            for (c in 0 until M) {
                val isDark = if (r - padQ in 0 until N && c - padQ in 0 until N) {
                    grid[r - padQ][c - padQ] == 1
                } else false
                val color = if (isDark) Color.BLACK else Color.WHITE
                for (dy in 0 until scale) {
                    for (dx in 0 until scale) {
                        bmp.setPixel(c * scale + dx, r * scale + dy, color)
                    }
                }
            }
        }
        return bmp
    }

    private fun startHttpServer() {
        threadPool.execute {
            try {
                serverSocket = ServerSocket().apply {
                    reuseAddress = true
                    bind(InetSocketAddress("0.0.0.0", PORT))
                }
                while (!isFinishing) {
                    val client = serverSocket?.accept() ?: break
                    threadPool.execute { handleClient(client) }
                }
            } catch (_: Exception) {}
        }
    }

    private fun readHeaderLine(input: InputStream): String? {
        val bos = ByteArrayOutputStream()
        while (true) {
            val b = input.read()
            if (b == -1) return if (bos.size() == 0) null else bos.toString("UTF-8").trimEnd('\r')
            if (b == '\n'.code) return bos.toString("UTF-8").trimEnd('\r')
            bos.write(b)
        }
    }

    private fun parseQueryParams(query: String?): Map<String, String> {
        if (query.isNullOrEmpty()) return emptyMap()
        val map = mutableMapOf<String, String>()
        for (pair in query.split("&")) {
            val idx = pair.indexOf('=')
            if (idx > 0) {
                val k = URLDecoder.decode(pair.substring(0, idx), "UTF-8")
                val v = URLDecoder.decode(pair.substring(idx + 1), "UTF-8")
                map[k] = v
            }
        }
        return map
    }

    private fun isSafeMediaPath(filepath: String): Boolean {
        return try {
            val target = File(filepath).canonicalFile
            if (!target.exists() || !target.isFile) return false
            FOLDER_MAP.values.flatten().any { dirPath ->
                val canonicalDir = File(dirPath).canonicalPath
                target.path.startsWith(canonicalDir + File.separator)
            }
        } catch (_: Exception) {
            false
        }
    }

    private fun formatSize(bytes: Long): String {
        return if (bytes >= 1024 * 1024) {
            String.format(Locale.US, "%.1f MB", bytes / (1024.0 * 1024.0))
        } else {
            String.format(Locale.US, "%d KB", bytes / 1024)
        }
    }

    private fun handleClient(socket: Socket) {
        socket.use { sock ->
            try {
                val inStream = BufferedInputStream(sock.getInputStream(), 65536)
                val outStream = BufferedOutputStream(sock.getOutputStream(), 65536)

                val requestLine = readHeaderLine(inStream) ?: return
                val parts = requestLine.split(" ")
                if (parts.size < 2) return
                val method = parts[0].uppercase(Locale.US)
                val fullUri = URI(parts[1])
                val path = fullUri.rawPath ?: "/"
                val query = parseQueryParams(fullUri.rawQuery)

                val headers = mutableMapOf<String, String>()
                while (true) {
                    val line = readHeaderLine(inStream) ?: break
                    if (line.isEmpty()) break
                    val colon = line.indexOf(':')
                    if (colon > 0) {
                        headers[line.substring(0, colon).trim().lowercase(Locale.US)] =
                            line.substring(colon + 1).trim()
                    }
                }

                if (method == "GET" && path == "/") {
                    val htmlBytes = HTML_PAGE.toByteArray(Charsets.UTF_8)
                    val respHeader = "HTTP/1.1 200 OK\r\n" +
                            "Content-Type: text/html; charset=utf-8\r\n" +
                            "Content-Length: ${htmlBytes.size}\r\n" +
                            "Connection: close\r\n\r\n"
                    outStream.write(respHeader.toByteArray(Charsets.UTF_8))
                    outStream.write(htmlBytes)
                    outStream.flush()

                } else if (method == "GET" && path == "/api/list") {
                    val folderKey = query["folder"] ?: "camera"
                    val offset = (query["offset"] ?: "0").toIntOrNull() ?: 0
                    val limit = (query["limit"] ?: "24").toIntOrNull() ?: 24
                    val targetDirs = FOLDER_MAP[folderKey] ?: FOLDER_MAP["camera"]!!

                    data class MediaItem(val mtime: Long, val size: Long, val name: String, val path: String, val ext: String)
                    val allFiles = mutableListOf<MediaItem>()

                    for (d in targetDirs) {
                        val dir = File(d)
                        if (!dir.isDirectory) continue
                        dir.listFiles()?.forEach { f ->
                            if (f.isFile) {
                                val dot = f.name.lastIndexOf('.')
                                val ext = if (dot >= 0) f.name.substring(dot).lowercase(Locale.US) else ""
                                if (ext in ALLOWED_EXTS) {
                                    allFiles.add(MediaItem(f.lastModified(), f.length(), f.name, f.absolutePath, ext))
                                }
                            }
                        }
                    }
                    allFiles.sortByDescending { it.mtime }
                    val sliced = allFiles.drop(offset).take(limit)
                    val sdf = SimpleDateFormat("MM/dd HH:mm", Locale.getDefault())

                    val itemsArr = JSONArray()
                    for (item in sliced) {
                        val obj = JSONObject()
                        obj.put("name", item.name)
                        obj.put("path", item.path)
                        obj.put("size", formatSize(item.size))
                        obj.put("date", sdf.format(Date(item.mtime)))
                        obj.put("is_video", item.ext in VIDEO_EXTS)
                        itemsArr.put(obj)
                    }
                    val rootObj = JSONObject()
                    rootObj.put("items", itemsArr)
                    rootObj.put("has_more", (offset + limit) < allFiles.size)
                    val jsonBytes = rootObj.toString().toByteArray(Charsets.UTF_8)

                    val respHeader = "HTTP/1.1 200 OK\r\n" +
                            "Content-Type: application/json; charset=utf-8\r\n" +
                            "Content-Length: ${jsonBytes.size}\r\n" +
                            "Connection: close\r\n\r\n"
                    outStream.write(respHeader.toByteArray(Charsets.UTF_8))
                    outStream.write(jsonBytes)
                    outStream.flush()

                } else if (method == "GET" && path == "/media") {
                    val filepath = query["path"] ?: ""
                    val isDl = query["dl"] == "1"
                    if (filepath.isEmpty() || !isSafeMediaPath(filepath)) {
                        outStream.write("HTTP/1.1 404 Not Found\r\nConnection: close\r\n\r\n".toByteArray())
                        outStream.flush()
                        return
                    }
                    val file = File(filepath)
                    val dot = file.name.lastIndexOf('.')
                    val ext = if (dot >= 0) file.name.substring(dot).lowercase(Locale.US) else ""
                    val mime = MIME_MAP[ext] ?: "application/octet-stream"
                    val fileSize = file.length()
                    val rangeHeader = headers["range"]

                    if (rangeHeader != null && !isDl && rangeHeader.startsWith("bytes=")) {
                        val rangeSpec = rangeHeader.removePrefix("bytes=").split("-")
                        val start = rangeSpec[0].toLongOrNull() ?: 0L
                        var end = if (rangeSpec.size > 1 && rangeSpec[1].isNotEmpty()) {
                            rangeSpec[1].toLongOrNull() ?: (fileSize - 1)
                        } else {
                            fileSize - 1
                        }
                        if (start >= fileSize || start > end) {
                            val err = "HTTP/1.1 416 Range Not Satisfiable\r\nContent-Range: bytes */$fileSize\r\nConnection: close\r\n\r\n"
                            outStream.write(err.toByteArray())
                            outStream.flush()
                            return
                        }
                        end = end.coerceAtMost(fileSize - 1)
                        val chunkLen = end - start + 1
                        val respHeader = "HTTP/1.1 206 Partial Content\r\n" +
                                "Content-Type: $mime\r\n" +
                                "Accept-Ranges: bytes\r\n" +
                                "Content-Range: bytes $start-$end/$fileSize\r\n" +
                                "Content-Length: $chunkLen\r\n" +
                                "Connection: close\r\n\r\n"
                        outStream.write(respHeader.toByteArray())
                        RandomAccessFile(file, "r").use { raf ->
                            raf.seek(start)
                            val buf = ByteArray(256 * 1024)
                            var rem = chunkLen
                            while (rem > 0) {
                                val r = raf.read(buf, 0, minOf(buf.size.toLong(), rem).toInt())
                                if (r <= 0) break
                                outStream.write(buf, 0, r)
                                rem -= r
                            }
                        }
                        outStream.flush()
                        return
                    }

                    val dispHeader = if (isDl) {
                        val encName = URLEncoder.encode(file.name, "UTF-8").replace("+", "%20")
                        appendLog("單檔傳至 iPhone: ${file.name}")
                        "Content-Disposition: attachment; filename*=UTF-8''$encName\r\n"
                    } else ""

                    val respHeader = "HTTP/1.1 200 OK\r\n" +
                            "Content-Type: $mime\r\n" +
                            "Accept-Ranges: bytes\r\n" +
                            "Content-Length: $fileSize\r\n" +
                            dispHeader +
                            "Connection: close\r\n\r\n"
                    outStream.write(respHeader.toByteArray())
                    FileInputStream(file).use { fis ->
                        val buf = ByteArray(256 * 1024)
                        while (true) {
                            val r = fis.read(buf)
                            if (r <= 0) break
                            outStream.write(buf, 0, r)
                        }
                    }
                    outStream.flush()

                } else if (method == "POST" && path == "/upload") {
                    SAVE_DIR.mkdirs()
                    val contentLength = (headers["content-length"] ?: "0").toLongOrNull() ?: 0L
                    val rawFilename = headers["x-filename"] ?: ""
                    val filename = if (rawFilename.isNotEmpty()) {
                        URLDecoder.decode(rawFilename, "UTF-8")
                    } else {
                        val ctype = (headers["content-type"] ?: "").lowercase(Locale.US)
                        val ext = when {
                            "heic" in ctype -> ".heic"
                            "png" in ctype -> ".png"
                            "video" in ctype || "quicktime" in ctype -> ".mov"
                            else -> ".jpg"
                        }
                        "IMG_" + SimpleDateFormat("yyyyMMdd_HHmmss_SSS", Locale.US).format(Date()) + ext
                    }

                    val safeName = File(filename).name
                    val dot = safeName.lastIndexOf('.')
                    val base = if (dot >= 0) safeName.substring(0, dot) else safeName
                    val ext = if (dot >= 0) safeName.substring(dot) else ""
                    var saveFile = File(SAVE_DIR, safeName)
                    var counter = 1
                    while (saveFile.exists()) {
                        saveFile = File(SAVE_DIR, "${base}_${counter}${ext}")
                        counter++
                    }

                    FileOutputStream(saveFile).use { fos ->
                        val buf = ByteArray(256 * 1024)
                        var rem = contentLength
                        while (rem > 0) {
                            val r = inStream.read(buf, 0, minOf(buf.size.toLong(), rem).toInt())
                            if (r <= 0) break
                            fos.write(buf, 0, r)
                            rem -= r
                        }
                    }

                    // 還原 iPhone 原始拍攝時間戳 (X-Mtime)
                    val rawMtime = headers["x-mtime"] ?: ""
                    val mtimeMs = rawMtime.toDoubleOrNull()?.toLong() ?: 0L
                    if (mtimeMs > 0L) {
                        saveFile.setLastModified(mtimeMs)
                    }

                    // 呼叫原生 Android MediaScanner 立即更新小米相簿
                    MediaScannerConnection.scanFile(
                        applicationContext,
                        arrayOf(saveFile.absolutePath),
                        null,
                        null
                    )
                    appendLog("已存入相簿: ${saveFile.name} (${formatSize(contentLength)})")

                    val resp = "HTTP/1.1 200 OK\r\nContent-Length: 2\r\nConnection: close\r\n\r\nOK"
                    outStream.write(resp.toByteArray())
                    outStream.flush()

                } else if (method == "POST" && path == "/api/zip") {
                    val contentLength = (headers["content-length"] ?: "0").toIntOrNull() ?: 0
                    val bodyBytes = ByteArray(contentLength)
                    var readTotal = 0
                    while (readTotal < contentLength) {
                        val r = inStream.read(bodyBytes, readTotal, contentLength - readTotal)
                        if (r <= 0) break
                        readTotal += r
                    }
                    val bodyStr = String(bodyBytes, 0, readTotal, Charsets.UTF_8)
                    val formParams = parseQueryParams(bodyStr)
                    val rawPaths = formParams["paths"] ?: "[]"
                    val jsonArr = JSONArray(rawPaths)
                    val validFiles = mutableListOf<File>()
                    for (i in 0 until jsonArr.length()) {
                        val p = jsonArr.optString(i, "")
                        if (p.isNotEmpty() && isSafeMediaPath(p)) {
                            validFiles.add(File(p))
                        }
                    }
                    if (validFiles.isEmpty()) {
                        outStream.write("HTTP/1.1 400 Bad Request\r\nConnection: close\r\n\r\n".toByteArray())
                        outStream.flush()
                        return
                    }

                    var tmpZip: File? = null
                    try {
                        tmpZip = File.createTempFile("xiaomi_pack_", ".zip", cacheDir)
                        val usedNames = mutableSetOf<String>()
                        ZipOutputStream(BufferedOutputStream(FileOutputStream(tmpZip))).use { zos ->
                            zos.setMethod(ZipOutputStream.DEFLATED)
                            zos.setLevel(Deflater.NO_COMPRESSION)
                            val buf = ByteArray(256 * 1024)
                            for (f in validFiles) {
                                var arcName = f.name
                                val dot = arcName.lastIndexOf('.')
                                val b = if (dot >= 0) arcName.substring(0, dot) else arcName
                                val e = if (dot >= 0) arcName.substring(dot) else ""
                                var idx = 1
                                while (arcName in usedNames) {
                                    arcName = "${b}_${idx}${e}"
                                    idx++
                                }
                                usedNames.add(arcName)
                                val entry = ZipEntry(arcName).apply { time = f.lastModified() }
                                zos.putNextEntry(entry)
                                FileInputStream(f).use { fis ->
                                    while (true) {
                                        val r = fis.read(buf)
                                        if (r <= 0) break
                                        zos.write(buf, 0, r)
                                    }
                                }
                                zos.closeEntry()
                            }
                        }

                        val zipSize = tmpZip.length()
                        val zipName = "Xiaomi_Photos_" + SimpleDateFormat("yyyyMMdd_HHmmss", Locale.US).format(Date()) + ".zip"
                        appendLog("打包至 iPhone: 共 ${validFiles.size} 個檔案 (${formatSize(zipSize)})")

                        val respHeader = "HTTP/1.1 200 OK\r\n" +
                                "Content-Type: application/zip\r\n" +
                                "Content-Length: $zipSize\r\n" +
                                "Content-Disposition: attachment; filename=\"$zipName\"\r\n" +
                                "Connection: close\r\n\r\n"
                        outStream.write(respHeader.toByteArray())
                        FileInputStream(tmpZip).use { fis ->
                            val buf = ByteArray(256 * 1024)
                            while (true) {
                                val r = fis.read(buf)
                                if (r <= 0) break
                                outStream.write(buf, 0, r)
                            }
                        }
                        outStream.flush()
                    } finally {
                        try { tmpZip?.delete() } catch (_: Exception) {}
                    }
                }
            } catch (_: Exception) {}
        }
    }

    companion object {
        private val HTML_PAGE = """<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>iPhone ⇄ 小米 雙向離線互傳</title>
    <style>
        * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
        body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; background: #f2f2f7; margin: 0; padding: 16px 16px 80px 16px; color: #1c1c1e; text-align: center; }
        .card { background: white; border-radius: 18px; padding: 20px; max-width: 500px; margin: 10px auto; box-shadow: 0 4px 14px rgba(0,0,0,0.07); }
        h2 { margin: 0 0 10px 0; font-size: 20px; }
        .badge { display: inline-block; background: #e5f9e7; color: #1e7e34; padding: 4px 10px; border-radius: 20px; font-size: 12px; font-weight: 600; margin-bottom: 8px; }
        .wakelock-tag { display: none; background: #fff3cd; color: #856404; padding: 4px 10px; border-radius: 20px; font-size: 12px; font-weight: 600; margin-bottom: 12px; }
        .wakelock-tag.active { display: inline-block; }
        .tabs { display: flex; background: #e5e5ea; border-radius: 12px; padding: 4px; margin-bottom: 18px; }
        .tab-btn { flex: 1; padding: 10px 6px; border: none; background: transparent; border-radius: 9px; font-size: 14px; font-weight: 600; color: #636366; cursor: pointer; transition: 0.2s; }
        .tab-btn.active { background: white; color: #007aff; box-shadow: 0 2px 6px rgba(0,0,0,0.08); }
        .panel { display: none; }
        .panel.active { display: block; }
        .btn { display: inline-block; background: #007aff; color: white; padding: 14px 20px; border-radius: 12px; font-size: 16px; font-weight: 600; border: none; width: 100%; cursor: pointer; margin-top: 10px; }
        .btn:disabled { background: #a1c6ea; cursor: not-allowed; }
        .btn-secondary { background: #f2f2f7; color: #007aff; border: 1px solid #d1d1d6; margin-top: 14px; padding: 11px; font-size: 14px; }
        input[type="file"] { display: none; }
        .file-label { display: block; border: 2px dashed #007aff; border-radius: 12px; padding: 30px 10px; color: #007aff; font-weight: 600; cursor: pointer; margin-bottom: 12px; background: #f8fbff; }
        #status { margin-top: 14px; font-size: 14px; color: #3a3a3c; word-break: break-all; line-height: 1.5; }
        progress { width: 100%; height: 12px; margin-top: 12px; display: none; }
        .folder-bar { display: flex; gap: 6px; overflow-x: auto; padding-bottom: 8px; margin-bottom: 10px; scrollbar-width: none; }
        .folder-bar::-webkit-scrollbar { display: none; }
        .folder-pill { white-space: nowrap; padding: 7px 12px; border-radius: 20px; border: 1px solid #d1d1d6; background: #f2f2f7; font-size: 13px; font-weight: 600; color: #3a3a3c; cursor: pointer; }
        .folder-pill.active { background: #007aff; color: white; border-color: #007aff; }
        .tip-box { background: #fff9e6; color: #856404; border-radius: 10px; padding: 8px 12px; font-size: 12px; text-align: left; margin-bottom: 10px; line-height: 1.5; }
        .batch-bar { display: flex; gap: 8px; margin-bottom: 12px; align-items: center; }
        .batch-btn { flex: 1; padding: 9px 10px; border-radius: 10px; font-size: 13px; font-weight: 600; border: 1px solid #d1d1d6; background: #f2f2f7; color: #1c1c1e; cursor: pointer; }
        .batch-btn.primary { flex: 1.6; background: #34c759; color: white; border-color: #34c759; }
        .batch-btn.primary:disabled { background: #a8e0b5; border-color: #a8e0b5; color: white; cursor: not-allowed; }
        .grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 10px; text-align: left; }
        .media-item { background: #f9f9fb; border: 2px solid #e5e5ea; border-radius: 12px; overflow: hidden; display: flex; flex-direction: column; transition: border-color 0.15s; }
        .media-item.selected { border-color: #007aff; background: #f0f7ff; }
        .thumb-wrap { position: relative; width: 100%; padding-top: 85%; background: #e5e5ea; overflow: hidden; cursor: pointer; }
        .thumb-wrap img, .thumb-wrap .video-placeholder { position: absolute; top: 0; left: 0; width: 100%; height: 100%; object-fit: cover; }
        .video-placeholder { display: flex; flex-direction: column; align-items: center; justify-content: center; background: #2c2c2e; color: white; font-size: 13px; }
        .check-badge { position: absolute; top: 8px; right: 8px; width: 28px; height: 28px; border-radius: 50%; background: rgba(0,0,0,0.45); border: 2px solid white; color: white; display: flex; align-items: center; justify-content: center; font-size: 15px; font-weight: bold; z-index: 5; cursor: pointer; }
        .media-item.selected .check-badge { background: #007aff; border-color: white; }
        .media-info { padding: 8px; font-size: 12px; cursor: pointer; }
        .media-name { font-weight: 600; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; color: #1c1c1e; }
        .media-meta { color: #8e8e93; font-size: 11px; margin-top: 2px; }
        .dl-btn { display: block; text-align: center; background: #eef6ff; color: #007aff; text-decoration: none; font-size: 12px; font-weight: 600; padding: 7px 4px; border-top: 1px solid #e5e5ea; }
        #floatingZipBar { display: none; position: fixed; bottom: 16px; left: 50%; transform: translateX(-50%); width: calc(100% - 32px); max-width: 480px; background: rgba(28,28,30,0.92); backdrop-filter: blur(8px); padding: 10px 14px; border-radius: 16px; box-shadow: 0 6px 20px rgba(0,0,0,0.25); z-index: 500; align-items: center; justify-content: space-between; gap: 10px; }
        #floatingZipBar.show { display: flex; }
        .float-text { color: white; font-size: 14px; font-weight: 600; text-align: left; }
        .float-btn { background: #34c759; color: white; border: none; padding: 10px 16px; border-radius: 10px; font-size: 14px; font-weight: 700; cursor: pointer; white-space: nowrap; }
        #lightbox { display: none; position: fixed; inset: 0; background: rgba(0,0,0,0.92); z-index: 999; flex-direction: column; justify-content: center; align-items: center; padding: 16px; }
        #lightbox.show { display: flex; }
        #lightbox img, #lightbox video { max-width: 100%; max-height: 75vh; border-radius: 8px; object-fit: contain; }
        .lb-hint { color: #fff; font-size: 14px; margin-top: 12px; font-weight: 600; background: rgba(255,255,255,0.18); padding: 6px 14px; border-radius: 20px; }
        .lb-actions { display: flex; gap: 12px; margin-top: 14px; width: 100%; max-width: 320px; }
        .lb-btn { flex: 1; padding: 11px; border-radius: 10px; font-size: 14px; font-weight: 600; text-align: center; text-decoration: none; border: none; cursor: pointer; }
        .lb-dl { background: #007aff; color: white; }
        .lb-close { background: #3a3a3c; color: white; }
    </style>
</head>
<body>
    <div class="card">
        <h2>📱 iPhone ⇄ 小米 雙向互傳</h2>
        <div>
            <span class="badge">⚡ 原檔無損直傳・自動保留原始拍攝時間與 EXIF</span>
            <br>
            <span id="wakeLockBadge" class="wakelock-tag">🔆 防休眠已啟動：傳輸期間螢幕保持常亮</span>
        </div>
        <div class="tabs">
            <button class="tab-btn active" onclick="switchTab('upload')">📤 傳送至小米</button>
            <button class="tab-btn" onclick="switchTab('download')">📥 從小米下載</button>
        </div>
        <div id="panel-upload" class="panel active">
            <div class="tip-box">
                💡 <b>100% 原檔無損秘訣：</b>選照片時點畫面底部<b>「選項」</b>，將格式改為<b>「目前格式」</b>並開啟<b>「位置」</b>，即可完整保留 HEIC 原檔與 GPS 座標！
            </div>
            <label class="file-label" for="fileInput" id="dropLabel">
                ➕ 點擊選擇 iPhone 照片或影片（可多選）
            </label>
            <input type="file" id="fileInput" multiple accept="image/*,video/*,.heic,.dng">
            <button class="btn" id="uploadBtn" onclick="uploadFiles()" disabled>開始離線高速傳輸</button>
            <progress id="progressBar" value="0" max="100"></progress>
            <div id="status">等待選擇檔案...</div>
        </div>
        <div id="panel-download" class="panel">
            <div class="folder-bar">
                <button class="folder-pill active" onclick="selectFolder('camera', this)">📷 小米相機</button>
                <button class="folder-pill" onclick="selectFolder('transfer', this)">🔄 互傳專區</button>
                <button class="folder-pill" onclick="selectFolder('screenshots', this)">🖼 螢幕截圖</button>
                <button class="folder-pill" onclick="selectFolder('download', this)">📂 下載資料夾</button>
            </div>
            <div class="tip-box">
                💡 <b>多張打包存回 iPhone 相簿步驟：</b><br>
                1. 點縮圖右上角「◯」勾選多張（或點全選）➔ 點<b>「📦 打包 ZIP 下載」</b>。<br>
                2. 下載後點 Safari 網址列旁「藍色箭頭」打開 ZIP，系統會自動解壓縮為資料夾，全選裡面照片按<b>「分享 ➔ 儲存影像」</b>即可一次存進相簿！
            </div>
            <div class="batch-bar">
                <button class="batch-btn" id="selectAllBtn" onclick="toggleSelectAll()">☑ 全選已載入</button>
                <button class="batch-btn primary" id="topZipBtn" onclick="downloadSelectedZip()" disabled>📦 打包 ZIP 下載 (0)</button>
            </div>
            <div id="galleryGrid" class="grid"></div>
            <button id="loadMoreBtn" class="btn btn-secondary" onclick="loadGallery(false)" style="display:none;">載入更多檔案...</button>
            <button class="btn btn-secondary" onclick="loadGallery(true)">🔄 重新整理列表</button>
        </div>
    </div>
    <div id="floatingZipBar">
        <div class="float-text" id="floatCountText">已勾選 0 個檔案</div>
        <button class="float-btn" onclick="downloadSelectedZip()">📦 打包 ZIP 下載</button>
    </div>
    <form id="zipForm" method="POST" action="/api/zip" style="display:none;">
        <input type="hidden" name="paths" id="zipPathsInput">
    </form>
    <div id="lightbox" onclick="closeLightbox(event)">
        <div id="lbContent"></div>
        <div class="lb-hint" id="lbHint">👆 長按上方圖片即可「儲存到相簿」</div>
        <div class="lb-actions">
            <a id="lbDownload" class="lb-btn lb-dl" href="#" download>⬇ 下載原檔</a>
            <button class="lb-btn lb-close" onclick="closeLightbox(null, true)">關閉</button>
        </div>
    </div>
    <script>
        var fileInput = document.getElementById('fileInput');
        var uploadBtn = document.getElementById('uploadBtn');
        var statusDiv = document.getElementById('status');
        var dropLabel = document.getElementById('dropLabel');
        var progressBar = document.getElementById('progressBar');
        var wakeLockBadge = document.getElementById('wakeLockBadge');
        var currentFolder = 'camera';
        var currentOffset = 0;
        var PAGE_LIMIT = 24;
        var selectedPaths = new Set();

        var nativeWakeLock = null;
        var keepAwakeVideo = null;
        var keepAwakeTimer = null;

        async function enableWakeLock() {
            wakeLockBadge.classList.add('active');
            if ('wakeLock' in navigator) {
                try {
                    nativeWakeLock = await navigator.wakeLock.request('screen');
                    return;
                } catch (e) {}
            }
            try {
                if (!keepAwakeVideo) {
                    var canvas = document.createElement('canvas');
                    canvas.width = 2;
                    canvas.height = 2;
                    var ctx = canvas.getContext('2d');
                    var toggle = false;
                    keepAwakeTimer = setInterval(function() {
                        ctx.fillStyle = toggle ? '#000000' : '#010101';
                        ctx.fillRect(0, 0, 2, 2);
                        toggle = !toggle;
                    }, 500);
                    keepAwakeVideo = document.createElement('video');
                    keepAwakeVideo.setAttribute('playsinline', '');
                    keepAwakeVideo.setAttribute('muted', '');
                    keepAwakeVideo.muted = true;
                    keepAwakeVideo.style.position = 'fixed';
                    keepAwakeVideo.style.width = '1px';
                    keepAwakeVideo.style.height = '1px';
                    keepAwakeVideo.style.opacity = '0.01';
                    keepAwakeVideo.style.pointerEvents = 'none';
                    keepAwakeVideo.style.bottom = '0';
                    keepAwakeVideo.style.right = '0';
                    document.body.appendChild(keepAwakeVideo);
                    if (canvas.captureStream) {
                        keepAwakeVideo.srcObject = canvas.captureStream(2);
                    }
                }
                await keepAwakeVideo.play();
            } catch (e) {}
        }

        async function disableWakeLock() {
            wakeLockBadge.classList.remove('active');
            if (nativeWakeLock) {
                try { await nativeWakeLock.release(); } catch (e) {}
                nativeWakeLock = null;
            }
            if (keepAwakeVideo) {
                try { keepAwakeVideo.pause(); } catch (e) {}
            }
        }

        function switchTab(tab) {
            document.querySelectorAll('.tab-btn').forEach(function(b, idx) {
                b.classList.toggle('active', (tab === 'upload' && idx === 0) || (tab === 'download' && idx === 1));
            });
            document.getElementById('panel-upload').classList.toggle('active', tab === 'upload');
            document.getElementById('panel-download').classList.toggle('active', tab === 'download');
            if (tab === 'download' && document.getElementById('galleryGrid').children.length === 0) {
                loadGallery(true);
            }
            updateSelectionUI();
        }

        fileInput.addEventListener('change', function() {
            var count = fileInput.files.length;
            if (count > 0) {
                dropLabel.textContent = '已選擇 ' + count + ' 個檔案';
                uploadBtn.disabled = false;
                statusDiv.textContent = '準備就緒，請點擊下方按鈕上傳';
            }
        });

        function sendSingleFile(file, index, total) {
            return new Promise(function(resolve) {
                var xhr = new XMLHttpRequest();
                xhr.open('POST', '/upload', true);
                xhr.setRequestHeader('X-Filename', encodeURIComponent(file.name));
                if (file.lastModified) {
                    xhr.setRequestHeader('X-Mtime', String(file.lastModified));
                }
                xhr.setRequestHeader('Content-Type', 'application/octet-stream');
                xhr.upload.onprogress = function(e) {
                    if (e.lengthComputable) {
                        var filePercent = Math.round((e.loaded / e.total) * 100);
                        var overallPercent = Math.round(((index + (e.loaded / e.total)) / total) * 100);
                        progressBar.value = overallPercent;
                        statusDiv.textContent = '正在傳送 (' + (index + 1) + '/' + total + '): ' + file.name + ' (' + filePercent + '%)';
                    }
                };
                xhr.onload = function() { resolve(xhr.status === 200); };
                xhr.onerror = function() { resolve(false); };
                xhr.send(file);
            });
        }

        async function uploadFiles() {
            var files = fileInput.files;
            if (!files.length) return;
            uploadBtn.disabled = true;
            progressBar.style.display = 'block';
            progressBar.value = 0;
            var successCount = 0;
            await enableWakeLock();
            try {
                for (var i = 0; i < files.length; i++) {
                    var ok = await sendSingleFile(files[i], i, files.length);
                    if (ok) successCount++;
                }
                progressBar.value = 100;
                statusDiv.textContent = '✅ 完成！成功傳送 ' + successCount + ' / ' + files.length + ' 個檔案至小米相簿（已保留原始時間）';
            } finally {
                await disableWakeLock();
                uploadBtn.disabled = false;
            }
        }

        function selectFolder(folder, btnEl) {
            currentFolder = folder;
            document.querySelectorAll('.folder-pill').forEach(function(p) { p.classList.remove('active'); });
            btnEl.classList.add('active');
            loadGallery(true);
        }

        function toggleSelectItem(path, cardEl, badgeEl, event) {
            if (event) event.stopPropagation();
            if (selectedPaths.has(path)) {
                selectedPaths.delete(path);
                cardEl.classList.remove('selected');
                badgeEl.textContent = '';
            } else {
                selectedPaths.add(path);
                cardEl.classList.add('selected');
                badgeEl.textContent = '✓';
            }
            updateSelectionUI();
        }

        function toggleSelectAll() {
            var cards = document.querySelectorAll('.media-item');
            if (!cards.length) return;
            var allSelected = Array.from(cards).every(function(c) { return selectedPaths.has(c.dataset.path); });
            cards.forEach(function(c) {
                var p = c.dataset.path;
                var badge = c.querySelector('.check-badge');
                if (allSelected) {
                    selectedPaths.delete(p);
                    c.classList.remove('selected');
                    badge.textContent = '';
                } else {
                    selectedPaths.add(p);
                    c.classList.add('selected');
                    badge.textContent = '✓';
                }
            });
            updateSelectionUI();
        }

        function updateSelectionUI() {
            var count = selectedPaths.size;
            var topZipBtn = document.getElementById('topZipBtn');
            var selectAllBtn = document.getElementById('selectAllBtn');
            var floatBar = document.getElementById('floatingZipBar');
            var floatText = document.getElementById('floatCountText');
            var isDownloadTab = document.getElementById('panel-download').classList.contains('active');
            topZipBtn.textContent = '📦 打包 ZIP 下載 (' + count + ')';
            topZipBtn.disabled = count === 0;
            floatText.textContent = '已勾選 ' + count + ' 個檔案';
            if (count > 0 && isDownloadTab) {
                floatBar.classList.add('show');
            } else {
                floatBar.classList.remove('show');
            }
            var cards = document.querySelectorAll('.media-item');
            var allSelected = cards.length > 0 && Array.from(cards).every(function(c) { return selectedPaths.has(c.dataset.path); });
            selectAllBtn.textContent = allSelected ? '✕ 取消全選' : '☑ 全選已載入';
        }

        function downloadSelectedZip() {
            if (selectedPaths.size === 0) return;
            enableWakeLock();
            setTimeout(function() { disableWakeLock(); }, 60000);
            var pathsArray = Array.from(selectedPaths);
            document.getElementById('zipPathsInput').value = JSON.stringify(pathsArray);
            document.getElementById('zipForm').submit();
        }

        async function loadGallery(reset) {
            if (reset === undefined) reset = true;
            var grid = document.getElementById('galleryGrid');
            var loadMoreBtn = document.getElementById('loadMoreBtn');
            if (reset) {
                currentOffset = 0;
                selectedPaths.clear();
                updateSelectionUI();
                grid.innerHTML = '<div style="grid-column:1/-1;text-align:center;padding:24px;color:#8e8e93;">讀取小米相簿中...</div>';
            }
            try {
                var res = await fetch('/api/list?folder=' + currentFolder + '&offset=' + currentOffset + '&limit=' + PAGE_LIMIT);
                var data = await res.json();
                if (reset) grid.innerHTML = '';
                if (data.items.length === 0 && currentOffset === 0) {
                    grid.innerHTML = '<div style="grid-column:1/-1;text-align:center;padding:28px;color:#8e8e93;">此資料夾目前沒有照片或影片</div>';
                    loadMoreBtn.style.display = 'none';
                    return;
                }
                data.items.forEach(function(item) {
                    var previewUrl = '/media?path=' + encodeURIComponent(item.path);
                    var dlUrl = '/media?path=' + encodeURIComponent(item.path) + '&dl=1';
                    var card = document.createElement('div');
                    card.className = 'media-item';
                    card.dataset.path = item.path;
                    if (selectedPaths.has(item.path)) {
                        card.classList.add('selected');
                    }
                    var thumbHtml = item.is_video
                        ? '<div class="video-placeholder"><span style="font-size:28px;">▶️</span><span style="margin-top:4px;">點擊預覽影片</span></div>'
                        : '<img src="' + previewUrl + '" loading="lazy" alt="' + item.name + '">';
                    card.innerHTML =
                        '<div class="thumb-wrap">' +
                            '<div class="check-badge">' + (selectedPaths.has(item.path) ? '✓' : '') + '</div>' +
                            thumbHtml +
                        '</div>' +
                        '<div class="media-info">' +
                            '<div class="media-name">' + item.name + '</div>' +
                            '<div class="media-meta">' + item.size + ' · ' + item.date + '</div>' +
                        '</div>' +
                        '<a class="dl-btn" href="' + dlUrl + '" download="' + item.name + '">⬇ 單檔下載</a>';
                    var thumbWrap = card.querySelector('.thumb-wrap');
                    var badge = card.querySelector('.check-badge');
                    var infoBox = card.querySelector('.media-info');
                    badge.addEventListener('click', function(e) { toggleSelectItem(item.path, card, badge, e); });
                    infoBox.addEventListener('click', function(e) { toggleSelectItem(item.path, card, badge, e); });
                    thumbWrap.addEventListener('click', function() { openLightbox(previewUrl, dlUrl, item.is_video); });
                    grid.appendChild(card);
                });
                currentOffset += data.items.length;
                loadMoreBtn.style.display = data.has_more ? 'block' : 'none';
                updateSelectionUI();
            } catch (e) {
                grid.innerHTML = '<div style="grid-column:1/-1;text-align:center;padding:20px;color:#d9534f;">讀取失敗，請先在小米 App 點擊橘色按鈕授予相簿權限</div>';
            }
        }

        function openLightbox(previewUrl, dlUrl, isVideo) {
            var lb = document.getElementById('lightbox');
            var content = document.getElementById('lbContent');
            var hint = document.getElementById('lbHint');
            var dlBtn = document.getElementById('lbDownload');
            dlBtn.href = dlUrl;
            if (isVideo) {
                content.innerHTML = '<video src="' + previewUrl + '" controls autoplay playsinline preload="metadata"></video>';
                hint.textContent = '🎬 支援直接播放與快轉・點下方「⬇ 下載原檔」可儲存';
            } else {
                content.innerHTML = '<img src="' + previewUrl + '">';
                hint.textContent = '👆 長按上方圖片即可「儲存到 iPhone 相簿」';
            }
            lb.classList.add('show');
        }

        function closeLightbox(e, force) {
            if (force || (e && e.target.id === 'lightbox')) {
                var lb = document.getElementById('lightbox');
                document.getElementById('lbContent').innerHTML = '';
                lb.classList.remove('show');
            }
        }
    </script>
</body>
</html>"""
    }
}
