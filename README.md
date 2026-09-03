# ==============================================================================
# ✍️ Tablet Support (OpenTabletDriver)
# This game has built-in integration configurations for OpenTabletDriver (OTD).
# * If you don't have it installed, the game will provide an official download option.
# * You can easily toggle Tablet Mode ON/OFF via the in-game Settings menu.
# ==============================================================================

import pygame
import random
import sys
import math
import webbrowser
import os

# Initialize Pygame and Mixer for Audio
pygame.init()
pygame.mixer.init()

# Game Setup
SCREEN_WIDTH = 800
SCREEN_HEIGHT = 600
screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
pygame.display.set_caption("Ucup ofs")
clock = pygame.time.Clock()

# Colors
BACKGROUND = (10, 10, 15)
MENU_BG = (15, 10, 25)
CIRCLE_COLOR = (255, 105, 180)    # Pink (Osu Standard)
DIVA_COLOR = (0, 220, 255)        # Light Blue / Cyan (Osu Diva)
APPROACH_COLOR = (255, 255, 255)  # White
TEXT_COLOR = (255, 255, 255)
ACCENT_COLOR = (0, 255, 200)      # Cyan
RED_COLOR = (255, 50, 50)
GREEN_COLOR = (50, 255, 50)

# Game Flow States
# "WELCOME" -> "ACCOUNT_CREATION" -> "MODE_SELECT" -> "MENU" -> "PLAYING" -> "RESULTS" -> "SETTINGS"
game_state = "WELCOME"  
game_mode = "STANDARD"  

# Account System Variables
username_input = ""
account_status = "Waiting for input..."
is_account_valid = False

# Gameplay Tracker Stats
score = 0
combo = 0
total_spawned_circles = 0
stat_hits = 0
stat_bad_timings = 0
stat_misses = 0
calculated_final_rank = "D"

# Editable Settings
circle_radius = 40       
approach_speed = 0.05    
bg_dim = 1.0             
tablet_mode = False      

# Fonts
title_font = pygame.font.SysFont("Arial", 54, bold=True)
font = pygame.font.SysFont("Arial", 24)
sub_font = pygame.font.SysFont("Arial", 18)

# ------------------------------------------------------------------------------
# ACCOUNT VALIDATION LOGIC
# ------------------------------------------------------------------------------
def check_account_validity(username):
    if len(username) < 3:
        return "INVALID: Username must be at least 3 characters!", False
    if " " in username:
        return "INVALID: Username cannot contain spaces!", False
    return "ACCOUNT VALID! Press ENTER to proceed.", True

# ------------------------------------------------------------------------------
# PERFORMANCE RANK CALCULATION
# ------------------------------------------------------------------------------
def calculate_performance_rank():
    if total_spawned_circles == 0:
        return "D"
    # Formula based on hit precision percentage
    accuracy = (stat_hits / total_spawned_circles) * 100
    
    if accuracy >= 100: return "SS"
    elif accuracy >= 95: return "S"
    elif accuracy >= 85: return "A"
    elif accuracy >= 70: return "B"
    elif accuracy >= 50: return "C"
    else: return "D"

# ------------------------------------------------------------------------------
# AUTOMATIC ASSET GENERATOR (Fallback placeholders if files don't exist yet)
# ------------------------------------------------------------------------------
if not os.path.exists("ucup.png"):
    temp_surf = pygame.Surface((200, 200), pygame.SRCALPHA)
    pygame.draw.circle(temp_surf, CIRCLE_COLOR, (100, 100), 90)
    pygame.draw.circle(temp_surf, (255, 255, 255), (65, 80), 12)
    pygame.draw.circle(temp_surf, (255, 255, 255), (135, 80), 12)
    pygame.draw.circle(temp_surf, (0, 0, 0), (65, 80), 5)
    pygame.draw.circle(temp_surf, (0, 0, 0), (135, 80), 5)
    pygame.draw.arc(temp_surf, (0, 0, 0), (60, 90, 80, 50), 3.14, 0, 4)
    pygame.image.save(temp_surf, "ucup.png")

ucup_img = pygame.image.load("ucup.png")
ucup_img = pygame.transform.scale(ucup_img, (180, 180))

welcome_played = False
def play_welcome_voice():
    global welcome_played
    if not welcome_played:
        if os.path.exists("welcome.mp3"):
            try:
                pygame.mixer.music.load("welcome.mp3")
                pygame.mixer.music.play()
            except:
                pass
        welcome_played = True

# ------------------------------------------------------------------------------
# GAME PLAYING ENTITY CLASS
# ------------------------------------------------------------------------------
class HitCircle:
    def __init__(self):
        self.x = random.randint(150, SCREEN_WIDTH - 150)
        self.y = random.randint(150, SCREEN_HEIGHT - 150)
        self.radius = circle_radius
        self.alive = True
        
        if game_mode == "STANDARD":
            self.approach_scale = 3.0
            self.approach_speed = approach_speed
        elif game_mode == "DIVA":
            self.start_dist = 200
            self.current_dist = self.start_dist
            self.fly_speed = approach_speed * 120  
            
    def update(self):
        if game_mode == "STANDARD":
            self.approach_scale -= self.approach_speed
            if self.approach_scale <= 1.0:
                self.alive = False
                return False
        elif game_mode == "DIVA":
            self.current_dist -= self.fly_speed
            if self.current_dist <= 0:
                self.alive = False
                return False
        return True

    def draw(self, surface):
        if not self.alive:
            return
        if game_mode == "STANDARD":
            current_approach_radius = int(self.radius * self.approach_scale)
            pygame.draw.circle(surface, APPROACH_COLOR, (self.x, self.y), current_approach_radius, 2)
            pygame.draw.circle(surface, CIRCLE_COLOR, (self.x, self.y), self.radius)
        elif game_mode == "DIVA":
            pygame.draw.circle(surface, (60, 60, 70), (self.x, self.y), self.radius, 2)
            note_x = int(self.x + math.cos(self.current_dist * 0.05) * self.current_dist)
            note_y = int(self.y + math.sin(self.current_dist * 0.05) * self.current_dist)
            pygame.draw.circle(surface, DIVA_COLOR, (note_x, note_y), int(self.radius * 0.8))

    def check_click(self, mouse_pos):
        distance = math.hypot(self.x - mouse_pos[0], self.y - mouse_pos[1])
        if distance <= self.radius:
            if game_mode == "STANDARD":
                if 1.0 <= self.approach_scale <= 1.5: return "HIT"
                else: return "BAD_TIMING"
            elif game_mode == "DIVA":
                if 0 <= self.current_dist <= 25: return "HIT"
                else: return "BAD_TIMING"
        return "MISS"

circles = []
spawn_timer = 0
spawn_rate = 60
welcome_timer = 0

# Main Game Loop
running = True
while running:
    mouse_pos = pygame.mouse.get_pos()
    
    # --- 1. STATE: WELCOME INTRO SCREEN ---
    if game_state == "WELCOME":
        screen.fill(BACKGROUND)
        play_welcome_voice()
        welcome_timer += 1
        pulse = int(math.sin(welcome_timer * 0.1) * 10)
        pulsed_img = pygame.transform.scale(ucup_img, (180 + pulse, 180 + pulse))
        
        screen.blit(pulsed_img, (SCREEN_WIDTH // 2 - pulsed_img.get_width() // 2, SCREEN_HEIGHT // 2 - 140))
        screen.blit(title_font.render("UCUP OFS", True, CIRCLE_COLOR), (SCREEN_WIDTH // 2 - 110, SCREEN_HEIGHT // 2 + 80))
        screen.blit(font.render("Press ANY KEY to Continue", True, TEXT_COLOR), (SCREEN_WIDTH // 2 - 120, SCREEN_HEIGHT // 2 + 160))
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT: running = False
            elif event.type in [pygame.KEYDOWN, pygame.MOUSEBUTTONDOWN]:
                game_state = "ACCOUNT_CREATION"

    # --- 2. STATE: ACCOUNT CREATION & CHECK VALIDITY ---
    elif game_state == "ACCOUNT_CREATION":
        screen.fill(MENU_BG)
        screen.blit(title_font.render("CREATE ACCOUNT", True, ACCENT_COLOR), (120, 80))
        screen.blit(font.render("Type Username:", True, TEXT_COLOR), (120, 200))
        
        # Input Box Graphic
        pygame.draw.rect(screen, (30, 30, 45), (120, 240, 400, 45), border_radius=5)
        pygame.draw.rect(screen, ACCENT_COLOR if is_account_valid else RED_COLOR, (120, 240, 400, 45), 2, border_radius=5)
        
        # Render text input
        screen.blit(font.render(username_input + "|", True, TEXT_COLOR), (135, 250))
        
        # Render validation check status text
        status_color = GREEN_COLOR if is_account_valid else RED_COLOR
        screen.blit(sub_font.render(f"System Check: {account_status}", True, status_color), (120, 300))
        screen.blit(sub_font.render("Press [ENTER] to confirm registration when valid.", True, TEXT_COLOR), (120, 450))

        for event in pygame.event.get():
            if event.type == pygame.QUIT: running = False
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_RETURN and is_account_valid:
                    game_state = "MODE_SELECT"
                elif event.key == pygame.K_BACKSPACE:
                    username_input = username_input[:-1]
                else:
                    if len(username_input) < 15 and event.unicode.isalnum(): # letters and numbers only
                        username_input += event.unicode
                
                # Check validity instantly on keystroke
                account_status, is_account_valid = check_account_validity(username_input)

    # --- 3. STATE: GAME MODES SELECT ---
    elif game_state == "MODE_SELECT":
        screen.fill(MENU_BG)
        screen.blit(title_font.render("SELECT GAME MODE", True, ACCENT_COLOR), (180, 80))
        
        pygame.draw.rect(screen, (30, 20, 40), (120, 220, 240, 180), border_radius=10)
        pygame.draw.rect(screen, (20, 35, 50), (440, 220, 240, 180), border_radius=10)
        screen.blit(font.render(" [1] OSU STANDARD", True, CIRCLE_COLOR), (130, 300))
        screen.blit(font.render(" [2] OSU DIVA STYLE", True, DIVA_COLOR), (450, 300))
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT: running = False
            elif event.type == pygame.KEYDOWN:
