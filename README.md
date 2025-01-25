<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Thiệp Cảm Ơn</title>
    <style>
        body {
            font-family: 'Arial', sans-serif;
            background: url('https://i.pinimg.com/originals/4e/c9/d7/4ec9d7d0b3a6b7c7d36a79c0d908b360.jpg') no-repeat center center fixed;
            background-size: cover;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            animation: fadeInBackground 2s ease-out;
        }

        @keyframes fadeInBackground {
            from {
                opacity: 0;
            }
            to {
                opacity: 1;
            }
        }

        .card {
            width: 500px;
            padding: 40px;
            background-color: rgba(27, 94, 32, 0.8);
            border-radius: 20px;
            box-shadow: 0 10px 15px rgba(0, 0, 0, 0.1);
            text-align: center;
            position: relative;
            color: #FFFFFF;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            animation: slideIn 1s ease-out, fadeInText 1s ease-in-out;
        }

        @keyframes slideIn {
            from {
                transform: translateY(30px);
                opacity: 0;
            }
            to {
                transform: translateY(0);
                opacity: 1;
            }
        }

        @keyframes fadeInText {
            from {
                opacity: 0;
                transform: translateX(-20px);
            }
            to {
                opacity: 1;
                transform: translateX(0);
            }
        }

        .content {
            font-size: 18px;
            line-height: 1.6;
            text-align: left;
            margin-left: 30px;
            margin-right: 30px;
            font-family: 'Comic Sans MS', sans-serif;
            margin-top: 20px;
            animation: fadeInText 1.5s ease-in-out;
        }

        .cat-img {
            width: 150px;
            height: 150px;
            object-fit: cover;
            border-radius: 15px;
            margin-top: 30px;
            box-shadow: 0 8px 12px rgba(0, 0, 0, 0.2);
            border: 5px solid white;
            background-color: #fff;
            transition: transform 0.3s ease;
        }

        .cat-img:hover {
            transform: scale(1.1);
        }

        .title {
            font-size: 20px;
            color: #FFD700;
            font-weight: bold;
            margin-bottom: 20px;
            text-align: center;
            display: flex;
            justify-content: center;
            align-items: center;
            position: relative;
            text-shadow: 2px 2px 5px rgba(0, 0, 0, 0.3);
        }

        .title::before {
            content: "🌿🌟";
            font-size: 30px;
            margin-right: 10px;
        }

        .title::after {
            content: "🌟🌿";
            font-size: 30px;
            margin-left: 10px;
        }

        .title span {
            font-size: 24px;
            color: white;
            animation: bounceTitle 1s infinite alternate;
        }

        @keyframes bounceTitle {
            0% {
                transform: translateY(0);
            }
            100% {
                transform: translateY(-10px);
            }
        }

        .card:hover {
            box-shadow: 0 15px 30px rgba(0, 0, 0, 0.3);
            transform: scale(1.05);
            transition: transform 0.3s ease, box-shadow 0.3s ease;
        }

        .cat-img {
            animation: rotateCat 4s infinite;
        }

        @keyframes rotateCat {
            0% {
                transform: rotate(0deg);
            }
            50% {
                transform: rotate(10deg);
            }
            100% {
                transform: rotate(0deg);
            }
        }
    </style>
</head>
<body>

    <div class="card">
        <div class="title">
            <span>Rosie Cám ơn Anh Trung!</span>
        </div>
        <img src="https://i.pinimg.com/originals/59/54/b4/5954b408c66525ad932faa693a647e3f.jpg" alt="Mèo dễ thương" class="cat-img">
        <div class="content">
            <p>Hello Anh Trung,</p>
            <p>Em cám ơn anh đã tặng cho em bàn phím mới. Em đã sử dụng nó để học cấp tốc một khóa HTML để thay lời cám ơn đến anh vì món quà tuyệt vời này.</p>
            <p>Chúc anh luôn vui vẻ và nhiều sức khỏe!</p>
        </div>
        <div class="content" style="text-align: center;">
            <p>Ký Tên,</p>
            <p><b>Rosie (không phải bạn thân, cũng không phải em gái anh Trung)</b></p>
        </div>
    </div>

</body>
</html>
