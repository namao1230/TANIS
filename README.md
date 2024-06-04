# TANIS
## 컴퓨터공학과 20210817 신현진
-----------



-----------

    ////////////////////////////////////////////////////////////
    // Headers
    ////////////////////////////////////////////////////////////
    #include <SFML/Graphics.hpp>
    #include <SFML/Audio.hpp>
    #include <cmath>
    #include <ctime>
    #include <cstdlib>
    
    #ifdef SFML_SYSTEM_IOS
    #include <SFML/Main.hpp>
    #endif
    
    // Function to get the resources directory
    std::string resourcesDir()
    {
    #ifdef SFML_SYSTEM_IOS
        return ""; // iOS의 경우 빈 문자열 반환
    #else
        return "resources/"; // 다른 플랫폼의 경우 resources/ 경로 반환
    #endif
    }
    
    // Function to generate a random color
    sf::Color getRandomColor() {
        return sf::Color(std::rand() % 256, std::rand() % 256, std::rand() % 256);
    }
    
    ////////////////////////////////////////////////////////////
    /// Entry point of application
    ///
    /// \return Application exit code
    ///
    ////////////////////////////////////////////////////////////
    int main()
    {
        std::srand(static_cast<unsigned int>(std::time(NULL))); // 난수 생성기 초기화
    
        // Define some constants
        const float pi = 3.14159f; // 파이 값
        const float gameWidth = 800; // 게임 창의 너비
        const float gameHeight = 600; // 게임 창의 높이
        sf::Vector2f paddleSize(25, 100); // 패들의 크기
        float ballRadius = 10.f; // 공의 반지름
    
        // Create the window of the application
        sf::RenderWindow window(sf::VideoMode(static_cast<unsigned int>(gameWidth), static_cast<unsigned int>(gameHeight), 32), "SFML Tennis",
            sf::Style::Titlebar | sf::Style::Close); // 게임 창 생성
        window.setVerticalSyncEnabled(true); // 수직 동기화 활성화
    
        // Load the sounds used in the game
        sf::SoundBuffer ballSoundBuffer;
        if (!ballSoundBuffer.loadFromFile(resourcesDir() + "ball.wav")) // 소리 파일 로드
            return EXIT_FAILURE;
        sf::Sound ballSound(ballSoundBuffer); // 공 소리 설정
    
        // Load the background music
        sf::Music backgroundMusic;
        if (!backgroundMusic.openFromFile(resourcesDir() + "sound.wav")) // 배경 음악 로드
            return EXIT_FAILURE;
        backgroundMusic.setLoop(true); // 배경 음악 반복 재생 설정
        backgroundMusic.play(); // 배경 음악 재생
    
        // Load the background texture
        sf::Texture backgroundTexture;
        if (!backgroundTexture.loadFromFile(resourcesDir() + "background.png")) // 배경 이미지 로드
            return EXIT_FAILURE;
    
        // Create a sprite for the background
        sf::Sprite backgroundSprite;
        backgroundSprite.setTexture(backgroundTexture); // 배경 이미지 설정
    
        // Create the left paddle
        sf::RectangleShape leftPaddle;
        leftPaddle.setSize(paddleSize - sf::Vector2f(3, 3)); // 패들 크기 설정
        leftPaddle.setOutlineThickness(3); // 테두리 두께 설정
        leftPaddle.setOutlineColor(sf::Color::Black); // 테두리 색상 설정
        leftPaddle.setFillColor(sf::Color(100, 100, 200)); // 패들 색상 설정
        leftPaddle.setOrigin(paddleSize / 2.f); // 패들 중심점 설정
    
        // Create the right paddle
        sf::RectangleShape rightPaddle;
        rightPaddle.setSize(paddleSize - sf::Vector2f(3, 3)); // 패들 크기 설정
        rightPaddle.setOutlineThickness(3); // 테두리 두께 설정
        rightPaddle.setOutlineColor(sf::Color::Black); // 테두리 색상 설정
        rightPaddle.setFillColor(sf::Color(200, 100, 100)); // 패들 색상 설정
        rightPaddle.setOrigin(paddleSize / 2.f); // 패들 중심점 설정
    
        // Create the ball
        sf::CircleShape ball;
        ball.setRadius(ballRadius - 3); // 공의 반지름 설정
        ball.setOutlineThickness(2); // 테두리 두께 설정
        ball.setOutlineColor(sf::Color::Black); // 테두리 색상 설정
        ball.setFillColor(sf::Color::White); // 공 색상 설정
        ball.setOrigin(ballRadius / 2, ballRadius / 2); // 공 중심점 설정
    
        // Load the text font
        sf::Font font;
        if (!font.loadFromFile(resourcesDir() + "tuffy.ttf")) // 폰트 로드
            return EXIT_FAILURE;
    
        // Initialize the pause message
        sf::Text pauseMessage;
        pauseMessage.setFont(font); // 폰트 설정
        pauseMessage.setCharacterSize(40); // 글자 크기 설정
        pauseMessage.setPosition(170.f, 200.f); // 메시지 위치 설정
        pauseMessage.setFillColor(sf::Color::White); // 메시지 색상 설정
    
    #ifdef SFML_SYSTEM_IOS
        pauseMessage.setString("Welcome to SFML Tennis!\nTouch the screen to start the game."); // iOS용 메시지
    #else
        pauseMessage.setString("Welcome to SFML Tennis!\n\nPress space to start the game."); // 다른 플랫폼용 메시지
    #endif
    
        // Define the paddles properties
        sf::Clock AITimer; // AI 타이머
        const sf::Time AITime = sf::seconds(0.1f); // AI 업데이트 주기
        const float paddleSpeed = 400.f; // 패들 속도
        float rightPaddleSpeed = 0.f; // 오른쪽 패들 속도
        const float ballSpeed = 400.f; // 공 속도
        float ballAngle = 0.f; // 공의 각도 (초기화)
    
        // Score tracking
        int playerScore = 0; // 플레이어 점수
        int computerScore = 0; // 컴퓨터 점수
        sf::Text scoreText;
        scoreText.setFont(font); // 폰트 설정
        scoreText.setCharacterSize(40); // 글자 크기 설정
        scoreText.setFillColor(sf::Color::White); // 색상 설정
        scoreText.setPosition(gameWidth / 2.f - 20.f, 20.f); // 점수 텍스트 위치 설정
    
        sf::Clock clock; // 게임 시계
        bool isPlaying = false; // 게임 상태 플래그
        while (window.isOpen())
        {
            // Handle events
            sf::Event event;
            while (window.pollEvent(event))
            {
                // Window closed or escape key pressed: exit
                if ((event.type == sf::Event::Closed) ||
                    ((event.type == sf::Event::KeyPressed) && (event.key.code == sf::Keyboard::Escape)))
                {
                    window.close(); // 창 닫기
                    break;
                }
    
                // Space key pressed: play
                if (((event.type == sf::Event::KeyPressed) && (event.key.code == sf::Keyboard::Space)) ||
                    (event.type == sf::Event::TouchBegan))
                {
                    if (!isPlaying)
                    {
                        // (re)start the game
                        isPlaying = true; // 게임 상태 갱신
                        clock.restart(); // 시계 재시작
    
                        // Reset the position of the paddles and ball
                        leftPaddle.setPosition(10.f + paddleSize.x / 2.f, gameHeight / 2.f); // 왼쪽 패들 위치 재설정
                        rightPaddle.setPosition(gameWidth - 10.f - paddleSize.x / 2.f, gameHeight / 2.f); // 오른쪽 패들 위치 재설정
                        ball.setPosition(gameWidth / 2.f, gameHeight / 2.f); // 공 위치 재설정
    
                        // Reset the ball angle
                        do
                        {
                            // Make sure the ball initial angle is not too much vertical
                            ballAngle = static_cast<float>(std::rand() % 360) * 2.f * pi / 360.f; // 공 각도 랜덤 설정
                        } while (std::abs(std::cos(ballAngle)) < 0.7f); // 너무 수직이 되지 않도록 조정
                    }
                }
    
                // Window size changed, adjust view appropriately
                if (event.type == sf::Event::Resized)
                {
                    sf::View view;
                    view.setSize(gameWidth, gameHeight); // 뷰 크기 설정
                    view.setCenter(gameWidth / 2.f, gameHeight / 2.f); // 뷰 중심 설정
                    window.setView(view); // 뷰 적용
                }
            }
    
            if (isPlaying)
            {
                float deltaTime = clock.restart().asSeconds(); // 프레임 간 경과 시간
    
                // Move the player's paddle
                if (sf::Keyboard::isKeyPressed(sf::Keyboard::Up) &&
                    (leftPaddle.getPosition().y - paddleSize.y / 2 > 5.f)) // 위쪽 방향키가 눌렸을 때
                {
                    leftPaddle.move(0.f, -paddleSpeed * deltaTime); // 패들 위로 이동
                }
                if (sf::Keyboard::isKeyPressed(sf::Keyboard::Down) &&
                    (leftPaddle.getPosition().y + paddleSize.y / 2 < gameHeight - 5.f)) // 아래쪽 방향키가 눌렸을 때
                {
                    leftPaddle.move(0.f, paddleSpeed * deltaTime); // 패들 아래로 이동
                }
    
                if (sf::Touch::isDown(0)) // 터치가 감지될 때
                {
                    sf::Vector2i pos = sf::Touch::getPosition(0); // 터치 위치
                    sf::Vector2f mappedPos = window.mapPixelToCoords(pos); // 터치 위치를 게임 좌표로 변환
                    leftPaddle.setPosition(leftPaddle.getPosition().x, mappedPos.y); // 패들 위치 설정
                }
    
                if (((rightPaddleSpeed < 0.f) && (rightPaddle.getPosition().y - paddleSize.y / 2 > 5.f)) ||
                    ((rightPaddleSpeed > 0.f) && (rightPaddle.getPosition().y + paddleSize.y / 2 < gameHeight - 5.f))) // 패들 이동 조건 체크
                {
                    rightPaddle.move(0.f, rightPaddleSpeed * deltaTime); // 패들 이동
                }
    
                // Update the computer's paddle direction according to the ball position
                if (AITimer.getElapsedTime() > AITime)
                {
                    AITimer.restart(); // AI 타이머 재시작
                    if (ball.getPosition().y + ballRadius > rightPaddle.getPosition().y + paddleSize.y / 2)
                        rightPaddleSpeed = paddleSpeed; // 공이 아래쪽에 있을 때 패들 속도 설정
                    else if (ball.getPosition().y - ballRadius < rightPaddle.getPosition().y - paddleSize.y / 2)
                        rightPaddleSpeed = -paddleSpeed; // 공이 위쪽에 있을 때 패들 속도 설정
                    else
                        rightPaddleSpeed = 0.f; // 공이 패들 범위 내에 있을 때 정지
                }
    
                // Check if the ball hits the top or bottom of the screen
                if (ball.getPosition().y - ballRadius < 0.f) {
                    // If the ball hits the top of the screen, reflect its vertical velocity
                    ballAngle = -ballAngle; // 공 각도 반전
                    ball.setPosition(ball.getPosition().x, ballRadius); // 공 위치 재설정
                    ballSound.play(); // 소리 재생
                }
                else if (ball.getPosition().y + ballRadius > gameHeight) {
                    // If the ball hits the bottom of the screen, reflect its vertical velocity
                    ballAngle = -ballAngle; // 공 각도 반전
                    ball.setPosition(ball.getPosition().x, gameHeight - ballRadius); // 공 위치 재설정
                    ballSound.play(); // 소리 재생
                }
                else {
                    // Move the ball
                    float factor = ballSpeed * deltaTime; // 공 이동 거리 계산
                    ball.move(std::cos(ballAngle) * factor, std::sin(ballAngle) * factor); // 공 이동
    
                    // Check collisions between the ball and the paddles
                    if (ball.getPosition().x - ballRadius < leftPaddle.getPosition().x + paddleSize.x / 2 &&
                        ball.getPosition().x - ballRadius > leftPaddle.getPosition().x &&
                        ball.getPosition().y + ballRadius >= leftPaddle.getPosition().y - paddleSize.y / 2 &&
                        ball.getPosition().y - ballRadius <= leftPaddle.getPosition().y + paddleSize.y / 2)
                    {
                        if (ball.getPosition().y > leftPaddle.getPosition().y)
                            ballAngle = pi - ballAngle + (std::rand() % 20) * pi / 180; // 공 각도 조정
                        else
                            ballAngle = pi - ballAngle - (std::rand() % 20) * pi / 180;
    
                        ball.setPosition(leftPaddle.getPosition().x + ballRadius + paddleSize.x / 2 + 0.1f, ball.getPosition().y); // 공 위치 재설정
                        ballSound.play(); // 소리 재생
    
                        // Change the color of the left paddle
                        leftPaddle.setFillColor(getRandomColor());
                    }
    
                    if (ball.getPosition().x + ballRadius > rightPaddle.getPosition().x - paddleSize.x / 2 &&
                        ball.getPosition().x + ballRadius < rightPaddle.getPosition().x &&
                        ball.getPosition().y + ballRadius >= rightPaddle.getPosition().y - paddleSize.y / 2 &&
                        ball.getPosition().y - ballRadius <= rightPaddle.getPosition().y + paddleSize.y / 2)
                    {
                        if (ball.getPosition().y > rightPaddle.getPosition().y)
                            ballAngle = pi - ballAngle + (std::rand() % 20) * pi / 180; // 공 각도 조정
                        else
                            ballAngle = pi - ballAngle - (std::rand() % 20) * pi / 180;
    
                        ball.setPosition(rightPaddle.getPosition().x - ballRadius - paddleSize.x / 2 - 0.1f, ball.getPosition().y); // 공 위치 재설정
                        ballSound.play(); // 소리 재생
    
                        // Change the color of the right paddle
                        rightPaddle.setFillColor(getRandomColor());
                    }
                }
    
                // Check if the ball is outside the screen and update the score
                if (ball.getPosition().x - ballRadius < 0.f)
                {
                    // Ball missed by the player
                    isPlaying = false; // 게임 상태 갱신
                    pauseMessage.setString("You lost!\nPress space to restart or\nescape to exit."); // 메시지 설정
                    computerScore++; // 컴퓨터 점수 증가
                }
                else if (ball.getPosition().x + ballRadius > gameWidth)
                {
                    // Ball missed by the computer
                    isPlaying = false; // 게임 상태 갱신
                    pauseMessage.setString("You won!\nPress space to restart or\nescape to exit."); // 메시지 설정
                    playerScore++; // 플레이어 점수 증가
                }
    
                // Update the score text
                scoreText.setString(std::to_string(playerScore) + " - " + std::to_string(computerScore)); // 점수 텍스트 갱신
            }
    
            // Clear the window
            window.clear(sf::Color(50, 50, 50)); // 창 지우기
    
            // Draw the background
            window.draw(backgroundSprite); // 배경 그리기
    
            if (isPlaying)
            {
                // Draw the paddles, ball, and score
                window.draw(leftPaddle); // 왼쪽 패들 그리기
                window.draw(rightPaddle); // 오른쪽 패들 그리기
                window.draw(ball); // 공 그리기
                window.draw(scoreText); // 점수 텍스트 그리기
            }
            else
            {
                // Draw the pause message
                window.draw(pauseMessage); // 일시정지 메시지 그리기
            }
    
            // Display things on screen
            window.display(); // 화면에 표시
        }
    
        return EXIT_SUCCESS; // 프로그램 종료
    }
