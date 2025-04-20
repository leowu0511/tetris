# Tetris 遊戲功能實現說明

本文檔詳細說明 Tetris 遊戲中各個功能的實現思路和對應的代碼片段。

## 1. 遊戲初始化與基本設置

### 1.1 遊戲視窗與基本設置

```python
# 初始化 Pygame
pygame.init()
pygame.mixer.init()  # 初始化音效系統

# 顏色定義
BLACK = (0, 0, 0)
WHITE = (255, 255, 255)
CYAN = (0, 255, 255)
YELLOW = (255, 255, 0)
MAGENTA = (255, 0, 255)
RED = (255, 0, 0)
GREEN = (0, 255, 0)
BLUE = (0, 0, 255)
ORANGE = (255, 165, 0)
GRAY = (40, 40, 40)  # 網格線顏色

# 遊戲設定
BLOCK_SIZE = 30
GRID_WIDTH = 10
GRID_HEIGHT = 20
SCREEN_WIDTH = BLOCK_SIZE * GRID_WIDTH
SCREEN_HEIGHT = BLOCK_SIZE * GRID_HEIGHT
NEXT_PIECE_SIZE = 4  # 下一個方塊預覽區域的大小
LOCK_DELAY = 1.0  # 方塊碰底後的延遲鎖定時間（秒）
MOVE_DELAY = 0.1  # 長按方向鍵時的移動延遲（秒）
MOVE_REPEAT_DELAY = 0.05  # 長按方向鍵時的重複移動延遲（秒）
```

### 1.2 方塊形狀定義

```python
# 方塊形狀定義
SHAPES = [
    [[1, 1, 1, 1]],  # I
    [[1, 1], [1, 1]],  # O
    [[1, 1, 1], [0, 1, 0]],  # T
    [[1, 1, 1], [1, 0, 0]],  # L
    [[1, 1, 1], [0, 0, 1]],  # J
    [[1, 1, 0], [0, 1, 1]],  # S
    [[0, 1, 1], [1, 1, 0]]   # Z
]

COLORS = [CYAN, YELLOW, MAGENTA, ORANGE, BLUE, GREEN, RED]
```

## 2. 遊戲核心功能

### 2.1 方塊生成

```python
def generate_piece(self):
    # 生成新的方塊
    shape_idx = random.randint(0, len(SHAPES) - 1)
    return {
        'shape': SHAPES[shape_idx],
        'color': COLORS[shape_idx]
    }

def new_piece(self):
    # 使用預覽的方塊作為當前方塊
    self.current_piece = {
        'shape': self.next_piece['shape'],
        'color': self.next_piece['color'],
        'x': GRID_WIDTH // 2 - len(self.next_piece['shape'][0]) // 2,
        'y': 0
    }
    # 生成下一個方塊
    self.next_piece = self.generate_piece()
    # 檢查遊戲是否結束
    if not self.is_valid_move(0, 0):
        self.game_over = True
    self.is_locking = False
    self.lock_time = 0
```

### 2.2 碰撞檢測

```python
def is_valid_move(self, dx, dy):
    # 檢查移動是否有效
    for y, row in enumerate(self.current_piece['shape']):
        for x, cell in enumerate(row):
            if cell:
                new_x = self.current_piece['x'] + x + dx
                new_y = self.current_piece['y'] + y + dy
                if (new_x < 0 or new_x >= GRID_WIDTH or 
                    new_y >= GRID_HEIGHT or 
                    (new_y >= 0 and self.grid[new_y][new_x])):
                    return False
    return True
```

### 2.3 方塊移動

```python
def move_piece(self, dx, dy):
    # 移動方塊
    if self.is_valid_move(dx, dy):
        self.current_piece['x'] += dx
        self.current_piece['y'] += dy
        # 如果移動成功，重置鎖定計時器
        self.is_locking = False
        self.lock_time = 0
        return True
    return False
```

### 2.4 方塊旋轉

```python
def rotate_piece(self):
    # 旋轉當前方塊
    old_shape = self.current_piece['shape']
    self.current_piece['shape'] = list(zip(*old_shape[::-1]))
    if not self.is_valid_move(0, 0):
        self.current_piece['shape'] = old_shape
    else:
        # 如果旋轉成功，重置鎖定計時器
        self.is_locking = False
        self.lock_time = 0
```

### 2.5 方塊鎖定與延遲

```python
def check_lock_condition(self):
    # 檢查方塊是否需要鎖定
    return not self.is_valid_move(0, 1)

def lock_piece(self):
    # 將當前方塊鎖定到網格中
    for y, row in enumerate(self.current_piece['shape']):
        for x, cell in enumerate(row):
            if cell:
                self.grid[self.current_piece['y'] + y][self.current_piece['x'] + x] = self.current_piece['color']
    play_sound('drop')  # 播放落地音效
    self.clear_lines()
    self.new_piece()
```

## 3. 遊戲機制

### 3.1 行消除

```python
def clear_lines(self):
    # 清除完整的行並計分
    lines_cleared = 0
    y = GRID_HEIGHT - 1
    while y >= 0:
        if all(self.grid[y]):
            # 創建消除特效
            self.create_line_clear_particles(y)
            lines_cleared += 1
            del self.grid[y]
            self.grid.insert(0, [0 for _ in range(GRID_WIDTH)])
        else:
            y -= 1
    if lines_cleared > 0:
        play_sound('clear')  # 播放消除音效
    self.score += lines_cleared * 100
```

### 3.2 粒子特效系統

```python
class Particle:
    def __init__(self, x, y, color):
        self.x = x
        self.y = y
        self.color = color
        self.lifetime = PARTICLE_LIFETIME
        angle = random.uniform(0, 2 * math.pi)
        speed = random.uniform(PARTICLE_SPEED * 0.5, PARTICLE_SPEED)
        self.vx = math.cos(angle) * speed
        self.vy = math.sin(angle) * speed
        self.size = random.randint(2, 4)

    def update(self, delta_time):
        self.x += self.vx * delta_time
        self.y += self.vy * delta_time
        self.lifetime -= delta_time
        return self.lifetime > 0

    def draw(self, screen):
        alpha = int(255 * (self.lifetime / PARTICLE_LIFETIME))
        color = (*self.color, alpha)
        surf = pygame.Surface((self.size * 2, self.size * 2), pygame.SRCALPHA)
        pygame.draw.circle(surf, color, (self.size, self.size), self.size)
        screen.blit(surf, (int(self.x - self.size), int(self.y - self.size)))

def create_line_clear_particles(self, y):
    # 為被消除的行的每個方塊創建粒子
    for x in range(GRID_WIDTH):
        if self.grid[y][x]:
            color = self.grid[y][x]
            center_x = x * BLOCK_SIZE + BLOCK_SIZE // 2
            center_y = y * BLOCK_SIZE + BLOCK_SIZE // 2
            for _ in range(PARTICLE_COUNT):
                self.particles.append(Particle(center_x, center_y, color))

def update_particles(self, delta_time):
    # 更新所有粒子
    self.particles = [p for p in self.particles if p.update(delta_time)]
```

### 3.3 音效系統

```python
# 音效設定
SOUNDS = {
    'drop': None,  # 方塊落地音效
    'clear': None  # 消除行音效
}

# 載入音效
def load_sounds():
    try:
        # 載入落地音效並調整音量
        drop_sound = pygame.mixer.Sound(os.path.join('sounds', 'drop.wav'))
        drop_sound.set_volume(0.5)  # 設置音量為50%
        SOUNDS['drop'] = drop_sound
        
        # 載入消除音效
        SOUNDS['clear'] = pygame.mixer.Sound(os.path.join('sounds', 'clear.wav'))
    except:
        print("Warning: Could not load sound files")

# 播放音效
def play_sound(sound_name):
    if SOUNDS[sound_name]:
        SOUNDS[sound_name].play()
```

## 4. 控制系統

### 4.1 按鍵處理

```python
def handle_key_press(self, key):
    # 處理按鍵按下事件
    if key == pygame.K_LEFT:
        self.move_piece(-1, 0)
        self.key_down_time = 0
        self.is_key_repeating = False
        self.repeat_key = pygame.K_LEFT
    elif key == pygame.K_RIGHT:
        self.move_piece(1, 0)
        self.key_down_time = 0
        self.is_key_repeating = False
        self.repeat_key = pygame.K_RIGHT
    elif key == pygame.K_DOWN:
        if self.move_piece(0, 1):
            self.key_down_time = 0
            self.is_key_repeating = False
            self.repeat_key = pygame.K_DOWN
    elif key == pygame.K_UP:
        self.rotate_piece()
    elif key == pygame.K_SPACE:
        # 空白鍵立即下落到底部
        while self.move_piece(0, 1):
            pass
        self.lock_piece()
```

### 4.2 長按功能實現

```python
def handle_key_repeat(self, delta_time):
    # 處理按鍵重複事件
    keys = pygame.key.get_pressed()
    if self.repeat_key and keys[self.repeat_key]:
        if not self.is_key_repeating:
            self.key_down_time += delta_time
            if self.key_down_time >= MOVE_DELAY:
                self.is_key_repeating = True
                self.key_repeat_time = 0
        else:
            self.key_repeat_time += delta_time
            if self.key_repeat_time >= MOVE_REPEAT_DELAY:
                self.key_repeat_time = 0
                if self.repeat_key == pygame.K_LEFT:
                    self.move_piece(-1, 0)
                elif self.repeat_key == pygame.K_RIGHT:
                    self.move_piece(1, 0)
                elif self.repeat_key == pygame.K_DOWN:
                    if not self.move_piece(0, 1):
                        self.is_locking = True  # 開始鎖定計時
                        self.lock_time = 0
    else:
        self.is_key_repeating = False
        self.key_down_time = 0
        self.repeat_key = None
```

## 5. 視覺效果

### 5.1 網格繪製

```python
def draw_grid(self):
    # 繪製網格線
    for x in range(GRID_WIDTH + 1):
        pygame.draw.line(self.screen, GRAY, 
                       (x * BLOCK_SIZE, 0), 
                       (x * BLOCK_SIZE, SCREEN_HEIGHT))
    for y in range(GRID_HEIGHT + 1):
        pygame.draw.line(self.screen, GRAY, 
                       (0, y * BLOCK_SIZE), 
                       (SCREEN_WIDTH, y * BLOCK_SIZE))
```

### 5.2 下一個方塊預覽

```python
def draw_next_piece(self):
    # 繪製下一個方塊預覽
    preview_x = 10
    preview_y = 50
    block_size = 20  # 預覽方塊的大小稍小一些
    
    # 繪製預覽區域背景
    pygame.draw.rect(self.screen, WHITE, 
                    (preview_x - 2, preview_y - 2, 
                     NEXT_PIECE_SIZE * block_size + 4, 
                     NEXT_PIECE_SIZE * block_size + 4), 2)
    
    # 計算方塊在預覽區域中的居中位置
    shape_width = len(self.next_piece['shape'][0])
    shape_height = len(self.next_piece['shape'])
    offset_x = (NEXT_PIECE_SIZE - shape_width) * block_size // 2
    offset_y = (NEXT_PIECE_SIZE - shape_height) * block_size // 2
    
    # 繪製下一個方塊
    for y, row in enumerate(self.next_piece['shape']):
        for x, cell in enumerate(row):
            if cell:
                pygame.draw.rect(self.screen, self.next_piece['color'],
                               (preview_x + offset_x + x * block_size,
                                preview_y + offset_y + y * block_size,
                                block_size - 1, block_size - 1))
```

### 5.3 遊戲結束文字渲染

```python
def render_text_with_outline(text, color, outline_color, y_pos, outline_size=2):
    # 先渲染黑色邊框
    outline_surface = self.font.render(text, True, outline_color)
    outline_rect = outline_surface.get_rect(center=(SCREEN_WIDTH // 2, y_pos))
    
    # 再渲染白色文字
    text_surface = self.font.render(text, True, color)
    text_rect = text_surface.get_rect(center=(SCREEN_WIDTH // 2, y_pos))
    
    # 繪製邊框
    for dx in range(-outline_size, outline_size + 1):
        for dy in range(-outline_size, outline_size + 1):
            if dx != 0 or dy != 0:  # 跳過中心點
                self.screen.blit(outline_surface, 
                              (outline_rect.x + dx, outline_rect.y + dy))
    
    # 繪製文字
    self.screen.blit(text_surface, text_rect)
    return text_rect
```

## 6. 遊戲主循環

### 6.1 主遊戲循環

```python
def run(self):
    while True:
        # 計算時間
        delta_time = self.clock.tick(60) / 1000.0
        self.fall_time += delta_time

        # 更新粒子
        self.update_particles(delta_time)

        # 處理事件
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                return
            if event.type == pygame.KEYDOWN:
                if not self.game_over:
                    self.handle_key_press(event.key)
                elif event.key == pygame.K_RETURN:
                    self.reset_game()

        # 處理按鍵重複
        if not self.game_over:
            self.handle_key_repeat(delta_time)

        # 自動下落
        if not self.game_over and not self.is_locking:
            if self.fall_time >= self.fall_speed:
                self.fall_time = 0
                if self.is_valid_move(0, 1):
                    self.current_piece['y'] += 1
                else:
                    self.is_locking = True  # 開始鎖定計時
                    self.lock_time = 0
        elif self.is_locking:
            # 檢查方塊是否仍然需要鎖定
            if self.check_lock_condition():
                self.lock_time += delta_time
                if self.lock_time >= LOCK_DELAY:
                    self.lock_piece()
            else:
                self.is_locking = False
                self.lock_time = 0

        self.draw()
```

## 7. 實現思路總結

### 7.1 方塊落地延遲鎖定

- 使用 `is_locking` 標記方塊是否正在等待鎖定
- 使用 `lock_time` 追蹤鎖定延遲時間
- 當方塊碰底時，開始鎖定計時
- 在延遲期間，玩家可以移動或旋轉方塊
- 移動或旋轉成功會重置鎖定計時器
- 使用 `check_lock_condition()` 確保方塊不會浮空鎖定

### 7.2 長按方向鍵連續移動

- 使用 `key_down_time` 追蹤按鍵按下的時間
- 使用 `key_repeat_time` 追蹤按鍵重複的時間
- 使用 `is_key_repeating` 標記是否正在重複按鍵
- 使用 `repeat_key` 記錄當前正在重複的按鍵
- 當按鍵按下超過 `MOVE_DELAY` 時間後，開始連續移動
- 連續移動的間隔為 `MOVE_REPEAT_DELAY`

### 7.3 粒子特效系統

- 使用 `Particle` 類管理單個粒子
- 每個粒子有自己的位置、速度、顏色和生命週期
- 在消除行時為每個方塊創建多個粒子
- 粒子會隨機方向和速度移動
- 粒子的透明度會隨著生命週期減少而降低
- 使用 `update_particles()` 方法更新所有粒子

### 7.4 網格線輔助定位

- 使用 `draw_grid()` 方法繪製灰色網格線
- 網格線幫助玩家更準確地判斷方塊位置
- 網格線顏色設置為深灰色，不會干擾遊戲視覺

### 7.5 下一個方塊預覽

- 在左上角顯示下一個方塊
- 使用白色邊框標示預覽區域
- 方塊在預覽區域中居中顯示
- 預覽方塊的大小稍小於遊戲中的方塊

### 7.6 遊戲結束提示

- 使用 `render_text_with_outline()` 方法渲染帶有黑色邊框的文字
- 遊戲結束時在中央顯示 "Game Over!" 文字
- 在下方顯示 "Press ENTER to Restart" 提示
- 文字使用白色，邊框使用黑色，增強可讀性 