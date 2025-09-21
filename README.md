# Shadow-Chaser-Game
# Shadow Chaser is a game which is developed using python function called "pygame". Pygame is a Python library used for making 2D games and multimedia applications.
import pygame
import sys
import random

pygame.init()

# Initialize Pygame mixer for music
pygame.mixer.init()
pygame.mixer.music.load("mission_impossible.mp3")
pygame.mixer.music.play(-1)  # loop forever

# Screen and grid setup
WIDTH, HEIGHT = 800, 600
TILE_SIZE = 40
ROWS, COLS = HEIGHT // TILE_SIZE, WIDTH // TILE_SIZE
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Shadow Chaser")

# Colors
BLACK = (0, 0, 0)
GRAY = (100, 100, 100)
WHITE = (255, 255, 255)
GREEN = (0, 255, 0)
YELLOW = (255, 255, 100)
RED = (255, 0, 0)
BLUE = (0, 100, 255)
ORANGE = (255, 200, 0)
PURPLE = (128, 0, 128)

# Player settings
player_radius = 15
base_light_radius = 120
player_speed = 5

# Fonts
font_big = pygame.font.SysFont(None, 48)
font_medium = pygame.font.SysFont(None, 30)
font_small = pygame.font.SysFont(None, 24)

# Tile meaning
power_ups = {4: "flashlight", 5: "light_upgrade", 6: "speed_boost"}

# Game state
kill_count = 0
lives = 2
current_level = 1


# Enemy class
class Enemy:
    def _init_(self, x, y, direction=1, speed=2):
        self.x = x
        self.y = y
        self.direction = direction
        self.speed = speed
        self.rect = pygame.Rect(int(self.x), int(self.y), TILE_SIZE, TILE_SIZE)

    def move(self):
        self.x += self.direction * self.speed
        if self.x < TILE_SIZE or self.x > WIDTH - TILE_SIZE * 2:
            self.direction *= -1
        self.rect.topleft = (int(self.x), int(self.y))

    def draw(self, px, py, light_radius):
        dist = ((self.x - px) ** 2 + (self.y - py) ** 2) ** 0.5
        if dist <= light_radius:
            pygame.draw.rect(screen, BLUE, self.rect)


# Welcome screen
def show_welcome_screen():
    clock = pygame.time.Clock()
    while True:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit(); sys.exit()
            if event.type == pygame.KEYDOWN and event.key == pygame.K_SPACE:
                return

        screen.fill(BLACK)
        title = font_big.render("SHADOW CHASER", True, YELLOW)
        screen.blit(title, title.get_rect(center=(WIDTH//2, 80)))

        lines = [
            "GAME RULES:",
            "• Use ARROW KEYS to move your character.",
            "• Only area within your LIGHT CIRCLE is visible.",
            "• Find the GREEN EXIT to complete levels.",
            "• GRAY blocks = Walls (impassable).",
            "• RED spikes = Instant death (restart level).",
            "• BLUE enemies = Death on contact (restart level).",
            "• ORANGE = Flashlight power-up (reveal map).",
            "• WHITE = Light upgrade (increase radius).",
            "• YELLOW = Speed boost (move faster).",
            "You have 2 lives total for the entire game.",
            "Press SPACE to start.",
        ]

        y = 140
        for line in lines:
            color = WHITE if ":" in line or line.startswith("•") else GRAY
            text = font_small.render(line, True, color)
            screen.blit(text, text.get_rect(center=(WIDTH//2, y)))
            y += 24

        pygame.display.flip()
        clock.tick(60)


# Game over screen
def show_game_over():
    clock = pygame.time.Clock()
    while True:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit(); sys.exit()
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_r:
                    return True
                if event.key == pygame.K_q:
                    pygame.quit(); sys.exit()

        screen.fill(BLACK)
        title = font_big.render("GAME OVER", True, RED)
        screen.blit(title, title.get_rect(center=(WIDTH//2, HEIGHT//2 - 30)))
        prompt = font_medium.render("Press R to Restart or Q to Quit", True, WHITE)
        screen.blit(prompt, prompt.get_rect(center=(WIDTH//2, HEIGHT//2 + 30)))
        pygame.display.flip()
        clock.tick(60)


# Maze generator
def generate_maze_with_walls(seed, level):
    random.seed(seed)
    maze = [[1 for _ in range(COLS)] for _ in range(ROWS)]
    stack = []
    start = (1, 1)
    maze[start[0]][start[1]] = 0
    stack.append(start)
    directions = [(0, 2), (2, 0), (0, -2), (-2, 0)]
    while stack:
        x, y = stack[-1]
        neighbors = []
        for dx, dy in directions:
            nx, ny = x + dx, y + dy
            if 1 <= nx < ROWS - 1 and 1 <= ny < COLS - 1 and maze[nx][ny] == 1:
                neighbors.append((nx, ny, dx // 2, dy // 2))
        if neighbors:
            nx, ny, wx, wy = random.choice(neighbors)
            maze[x + wx][y + wy] = 0
            maze[nx][ny] = 0
            stack.append((nx, ny))
        else:
            stack.pop()

    exit_candidates = []
    for i in range(COLS):
        if maze[1][i] == 0 and (1, i) != start:
            exit_candidates.append((1, i))
        if maze[ROWS - 2][i] == 0 and (ROWS - 2, i) != start:
            exit_candidates.append((ROWS - 2, i))
    for i in range(ROWS):
        if maze[i][1] == 0 and (i, 1) != start:
            exit_candidates.append((i, 1))
        if maze[i][COLS - 2] == 0 and (i, COLS - 2) != start:
            exit_candidates.append((i, COLS - 2))

    if exit_candidates:
        ex, ey = random.choice(exit_candidates)
    else:
        ex, ey = ROWS - 2, COLS - 2
    maze[ex][ey] = 2

    open_spaces = [(r, c) for r in range(1, ROWS - 1) for c in range(1, COLS - 1)
                   if maze[r][c] == 0 and (r, c) != start and (r, c) != (ex, ey)]
    random.shuffle(open_spaces)

    if level >= 2:
        num_powerups = min(3 + level // 3, len(open_spaces))
        for _ in range(num_powerups):
            if not open_spaces:
                break
            r, c = open_spaces.pop()
            maze[r][c] = random.choice([4, 5, 6])

    if level >= 2:
        num_enemies = min(2 + level, len(open_spaces))
        for _ in range(num_enemies):
            if not open_spaces:
                break
            r, c = open_spaces.pop()
            maze[r][c] = 7

    return maze


# Maze drawer
def draw_maze(maze, px, py, light_radius, reveal_map=False):
    for r, row in enumerate(maze):
        for c, tile in enumerate(row):
            x, y = c * TILE_SIZE, r * TILE_SIZE
            dist = ((x + TILE_SIZE // 2 - px) ** 2 + (y + TILE_SIZE // 2 - py) ** 2) ** 0.5
            if reveal_map or dist < light_radius + TILE_SIZE:
                if tile == 1:
                    pygame.draw.rect(screen, GRAY, (x, y, TILE_SIZE, TILE_SIZE))
                elif tile == 2:
                    pygame.draw.rect(screen, GREEN, (x, y, TILE_SIZE, TILE_SIZE))
                elif tile == 3:
                    pygame.draw.rect(screen, RED, (x, y, TILE_SIZE, TILE_SIZE))
                elif tile == 4:
                    pygame.draw.rect(screen, ORANGE, (x, y, TILE_SIZE, TILE_SIZE))
                elif tile == 5:
                    pygame.draw.rect(screen, WHITE, (x, y, TILE_SIZE, TILE_SIZE))
                elif tile == 6:
                    pygame.draw.rect(screen, YELLOW, (x, y, TILE_SIZE, TILE_SIZE))


# Play one level
def play_level(level_idx):
    global kill_count, lives, current_level
    maze = generate_maze_with_walls(level_idx * 1000 + current_level, current_level)
    px = 1 * TILE_SIZE + TILE_SIZE // 2
    py = 1 * TILE_SIZE + TILE_SIZE // 2
    light_radius = base_light_radius
    speed = player_speed
    flashlight_timer = 0
    speed_timer = 0
    enemies = []
    enemy_speed = 2 + max(0, current_level - 3)

    # Add 60-second timer (60 FPS)
    level_timer = 60 * 60  

    for r, row in enumerate(maze):
        for c, tile in enumerate(row):
            if tile == 7:
                enemies.append(Enemy(c * TILE_SIZE, r * TILE_SIZE, random.choice([-1, 1]), enemy_speed))

    clock = pygame.time.Clock()
    running = True
    while running:
        clock.tick(60)
        level_timer -= 1  # decrease timer each frame

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit(); sys.exit()

        keys = pygame.key.get_pressed()
        dx = dy = 0
        if keys[pygame.K_UP]: dy = -speed
        if keys[pygame.K_DOWN]: dy = speed
        if keys[pygame.K_LEFT]: dx = -speed
        if keys[pygame.K_RIGHT]: dx = speed

        new_px = px + dx
        new_col = int(new_px // TILE_SIZE)
        row_at_player = int(py // TILE_SIZE)
        if 0 <= new_col < COLS and maze[row_at_player][new_col] != 1:
            px = new_px

        new_py = py + dy
        new_row = int(new_py // TILE_SIZE)
        col_at_player = int(px // TILE_SIZE)
        if 0 <= new_row < ROWS and maze[new_row][col_at_player] != 1:
            py = new_py

        player_rect = pygame.Rect(int(px - player_radius), int(py - player_radius),
                                  player_radius * 2, player_radius * 2)
        cur_col = int(px // TILE_SIZE)
        cur_row = int(py // TILE_SIZE)

        screen.fill(BLACK)
        draw_maze(maze, px, py, light_radius, flashlight_timer > 0)
        pygame.draw.circle(screen, YELLOW, (int(px), int(py)), player_radius)

        # HUD
        screen.blit(font_medium.render(f'Level: {current_level}', True, YELLOW), (10, 10))
        screen.blit(font_medium.render(f'Lives: {lives}', True, GREEN), (10, 40))
        screen.blit(font_medium.render(f'Deaths: {kill_count}', True, WHITE), (10, 70))

        # Show countdown timer
        seconds_left = max(0, level_timer // 60)
        screen.blit(font_medium.render(f'Time: {seconds_left}s', True, RED), (10, 100))

        # Time over -> restart level
        if level_timer <= 0:
            return False

        for enemy in enemies:
            enemy.move()
            enemy.draw(px, py, light_radius)
            if player_rect.colliderect(enemy.rect):
                kill_count += 1
                lives -= 1
                pygame.time.delay(800)
                if lives <= 0:
                    show_game_over()
                    kill_count = 0
                    lives = 2
                    current_level = 1
                    return False
                else:
                    return False

        if maze[cur_row][cur_col] in power_ups:
            ptype = power_ups[maze[cur_row][cur_col]]
            if ptype == "flashlight":
                flashlight_timer = 120
            elif ptype == "light_upgrade":
                light_radius += 65
            elif ptype == "speed_boost":
                speed = player_speed * 2
                speed_timer = 180
            maze[cur_row][cur_col] = 0

        if flashlight_timer > 0:
            flashlight_timer -= 1
        if speed_timer > 0:
            speed_timer -= 1
        else:
            speed = player_speed

        if flashlight_timer == 0:
            darkness = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
            darkness.fill((0, 0, 0, 255))
            pygame.draw.circle(darkness, (0, 0, 0, 0), (int(px), int(py)), light_radius)
            screen.blit(darkness, (0, 0))

        if maze[cur_row][cur_col] == 3:
            lives -= 1
            pygame.time.delay(800)
            if lives <= 0:
                show_game_over()
                kill_count = 0
                lives = 2
                current_level = 1
                return False
            else:
                return False

        if maze[cur_row][cur_col] == 2:
            pygame.time.delay(800)
            return True

        pygame.display.flip()


# Level transition screen
def show_level_transition():
    clock = pygame.time.Clock()
    start_time = pygame.time.get_ticks()
    while pygame.time.get_ticks() - start_time < 1500:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit(); sys.exit()
        screen.fill(BLACK)
        text = font_big.render(f"LEVEL {current_level}", True, YELLOW)
        screen.blit(text, text.get_rect(center=(WIDTH//2, HEIGHT//2)))
        pygame.display.flip()
        clock.tick(60)


# Main loop
show_welcome_screen()
while True:
    show_level_transition()
    completed = False
    while not completed:
        completed = play_level(current_level - 1)
    current_level += 1
