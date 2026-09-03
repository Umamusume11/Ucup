import pygame
import random
import sys
import math

# Initialize Pygame
pygame.init()

# Game Setup
SCREEN_WIDTH = 800
SCREEN_HEIGHT = 600
screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
pygame.display.set_caption("Ucup - Mini Osu! Clone")
clock = pygame.time.Clock()

# Colors
BACKGROUND = (10, 10, 15)
CIRCLE_COLOR = (255, 105, 180) # Pink
APPROACH_COLOR = (255, 255, 255) # White
TEXT_COLOR = (255, 255, 255)

# Game Variables
score = 0
combo = 0
font = pygame.font.SysFont("Arial", 24)

# Hit Circle Class
class HitCircle:
    def __init__(self):
        self.x = random.randint(100, SCREEN_WIDTH - 100)
        self.y = random.randint(100, SCREEN_HEIGHT - 100)
        self.radius = 40
        self.approach_scale = 3.0 # Starts 3x bigger
        self.approach_speed = 0.05 # How fast the ring shrinks
        self.alive = True

    def update(self):
        # Shrink the approach circle
        self.approach_scale -= self.approach_speed
        if self.approach_scale <= 1.0:
            # Player missed the timing window
            self.alive = False
            return False
        return True

    def draw(self, surface):
        if not self.alive:
            return
        # Draw approach circle (outer ring)
        current_approach_radius = int(self.radius * self.approach_scale)
        pygame.draw.circle(surface, APPROACH_COLOR, (self.x, self.y), current_approach_radius, 2)
        # Draw hit circle (inner target)
        pygame.draw.circle(surface, CIRCLE_COLOR, (self.x, self.y), self.radius)

    def check_click(self, mouse_pos):
        # Calculate distance between mouse and circle center
        distance = math.hypot(self.x - mouse_pos[0], self.y - mouse_pos[1])
        if distance <= self.radius:
            # Perfect timing is when approach_scale is closest to 1.0
            if 1.0 <= self.approach_scale <= 1.5:
                return "HIT"
            else:
                return "BAD_TIMING"
        return "MISS"

# Manage circles
circles = []
spawn_timer = 0
spawn_rate = 60 # Spawn every 60 frames (~1 second)

# Main Game Loop
running = True
while running:
    screen.fill(BACKGROUND)
    mouse_pos = pygame.mouse.get_pos()
    
    # Event Handling
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
            
        elif event.type == pygame.MOUSEBUTTONDOWN:
            if event.button == 1: # Left click
                clicked_circle = False
                for circle in circles:
                    result = circle.check_click(mouse_pos)
                    if result == "HIT":
                        score += 300 * (combo + 1)
                        combo += 1
                        circle.alive = False
                        clicked_circle = True
                        break
                    elif result == "BAD_TIMING":
                        score += 50
                        combo = 0
                        circle.alive = False
                        clicked_circle = True
                        break
                if not clicked_circle:
                    combo = 0 # Missed completely

    # Spawn circles
    spawn_timer += 1
    if spawn_timer >= spawn_rate:
        circles.append(HitCircle())
        spawn_timer = 0

    # Update and Draw circles
    for circle in circles[:]:
        if not circle.update() or not circle.alive:
            if not circle.alive and circle.approach_scale <= 1.0:
                combo = 0 # Reset combo on timeout miss
            circles.remove(circle)
        else:
            circle.draw(screen)

    # UI text
    score_text = font.render(f"Score: {score}", True, TEXT_COLOR)
    combo_text = font.render(f"Combo: {combo}x", True, TEXT_COLOR)
    screen.blit(score_text, (20, 20))
    screen.blit(combo_text, (20, 50))

    pygame.display.flip()
    clock.tick(60) # 60 FPS

pygame.quit()
sys.exit()

