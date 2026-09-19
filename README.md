Gold Vibes
import pygame
import random
import json
import os
from dataclasses import dataclass

# ============================================================
# GOLD VIBES - Python/Pygame prototype
# Clean, fictional open-world career game.
# ============================================================

pygame.init()

WIDTH, HEIGHT = 1100, 700
FPS = 60
SCREEN = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Gold Vibes")
CLOCK = pygame.time.Clock()

FONT = pygame.font.Font(None, 28)
SMALL = pygame.font.Font(None, 22)
BIG = pygame.font.Font(None, 42)
TITLE = pygame.font.Font(None, 70)

SAVE_FILE = "gold_vibes_save.json"

# ---------- Colors ----------
WHITE = (245, 245, 245)
BLACK = (20, 20, 20)
GRAY = (95, 100, 108)
LIGHT_GRAY = (180, 185, 190)
ROAD = (55, 58, 64)
GRASS = (55, 125, 65)
SIDEWALK = (150, 150, 145)
GOLD = (238, 190, 45)
RED = (205, 65, 65)
BLUE = (65, 130, 220)
GREEN = (65, 180, 90)
BUILDING = (120, 105, 95)
WINDOW = (75, 145, 185)
CAR_COLORS = [(180, 50, 50), (50, 100, 190), (230, 190, 50),
              (70, 70, 75), (235, 235, 235), (50, 150, 90)]

WORLD_W, WORLD_H = 2400, 1800


# ---------- Helpers ----------
def clamp(value, low, high):
    return max(low, min(high, value))


def draw_text(surface, text, font, color, x, y):
    surface.blit(font.render(str(text), True, color), (x, y))


def distance(a, b):
    return ((a[0] - b[0]) ** 2 + (a[1] - b[1]) ** 2) ** 0.5


# ---------- Player ----------
@dataclass
class Player:
    x: float = 450
    y: float = 450
    speed: float = 240
    money: int = 500
    bank: int = 1000
    health: int = 100
    energy: int = 100
    respect: int = 0
    heat: int = 0
    rank: int = 1
    vehicle_id: int | None = None

    def rect(self):
        return pygame.Rect(int(self.x - 14), int(self.y - 14), 28, 28)

    def update_rank(self):
        new_rank = 1 + self.respect // 100
        self.rank = min(new_rank, 10)


# ---------- Cars ----------
@dataclass
class Car:
    x: float
    y: float
    color: tuple
    name: str
    price: int
    speed: float
    owned: bool = False
    occupied: bool = False

    def rect(self):
        return pygame.Rect(int(self.x - 25), int(self.y - 14), 50, 28)


cars = [
    Car(650, 450, CAR_COLORS[0], "City Runner", 0, 360, True),
    Car(900, 450, CAR_COLORS[1], "Blue Comet", 2500, 430),
    Car(1250, 850, CAR_COLORS[2], "Gold Cruiser", 7000, 500),
]


# ---------- Buildings ----------
buildings = [
    pygame.Rect(80, 80, 260, 180),
    pygame.Rect(420, 70, 240, 190),
    pygame.Rect(780, 70, 300, 190),
    pygame.Rect(1200, 70, 300, 190),
    pygame.Rect(1650, 70, 300, 190),

    pygame.Rect(80, 500, 260, 230),
    pygame.Rect(430, 520, 230, 220),
    pygame.Rect(800, 500, 300, 230),
    pygame.Rect(1200, 500, 270, 230),
    pygame.Rect(1600, 500, 320, 230),

    pygame.Rect(80, 1000, 260, 220),
    pygame.Rect(430, 980, 250, 240),
    pygame.Rect(800, 1010, 310, 210),
    pygame.Rect(1220, 990, 260, 240),
    pygame.Rect(1600, 1000, 300, 220),

    pygame.Rect(80, 1450, 270, 200),
    pygame.Rect(450, 1420, 260, 220),
    pygame.Rect(850, 1430, 290, 210),
    pygame.Rect(1250, 1420, 250, 220),
    pygame.Rect(1650, 1420, 300, 220),
]

# ---------- Important places ----------
BANK = pygame.Rect(100, 330, 180, 100)
SHOP = pygame.Rect(350, 330, 180, 100)
GARAGE = pygame.Rect(600, 330, 180, 100)
CLUB = pygame.Rect(850, 330, 180, 100)
HOME = pygame.Rect(1100, 330, 180, 100)


# ---------- NPCs ----------
class NPC:
    def __init__(self, x, y, name):
        self.x = x
        self.y = y
        self.name = name
        self.direction = random.choice([-1, 1])
        self.timer = random.uniform(1, 4)

    def update(self, dt):
        self.timer -= dt
        if self.timer <= 0:
            self.direction = random.choice([-1, 1])
            self.timer = random.uniform(1, 4)

        self.x += self.direction * 35 * dt
        self.x = clamp(self.x, 20, WORLD_W - 20)

    def draw(self, surface, camera):
        sx = int(self.x - camera[0])
        sy = int(self.y - camera[1])
        pygame.draw.circle(surface, (225, 175, 130), (sx, sy - 9), 8)
        pygame.draw.rect(surface, BLUE, (sx - 8, sy, 16, 20))


npc_names = ["Alex", "Sam", "Jordan", "Taylor", "Morgan", "Riley", "Casey"]
npcs = [
    NPC(300 + i * 230, 430 + (i % 3) * 70, random.choice(npc_names))
    for i in range(14)
]


# ---------- Missions ----------
MISSIONS = [
    {
        "name": "First Steps",
        "description": "Meet your contact at the downtown shop.",
        "reward": 400,
        "respect": 15,
        "target": "shop",
    },
    {
        "name": "Late Delivery",
        "description": "Drive to the garage and deliver the package.",
        "reward": 800,
        "respect": 25,
        "target": "garage",
    },
    {
        "name": "Quiet Business",
        "description": "Visit the club and speak with the manager.",
        "reward": 1200,
        "respect": 35,
        "target": "club",
    },
    {
        "name": "New Connections",
        "description": "Return home after meeting your contacts.",
        "reward": 1800,
        "respect": 50,
        "target": "home",
    },
]


class Game:
    def __init__(self):
        self.player = Player()
        self.running = True
        self.paused = False
        self.message = "Welcome to Gold Vibes."
        self.message_timer = 4
        self.mission_index = 0
        self.day = 1
        self.hour = 8
        self.minute = 0
        self.camera_x = 0
        self.camera_y = 0
        self.in_car = False
        self.active_car = None
        self.show_help = False
        self.show_career = False

    # ---------- Saving ----------
    def save(self):
        data = {
            "player": self.player.__dict__,
            "mission_index": self.mission_index,
            "day": self.day,
            "hour": self.hour,
            "minute": self.minute,
            "cars": [
                {
                    "x": c.x, "y": c.y, "color": c.color,
                    "name": c.name, "price": c.price,
                    "speed": c.speed, "owned": c.owned,
                    "occupied": False
                }
                for c in cars
            ]
        }

        with open(SAVE_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2)

        self.notify("Game saved.")

    def load(self):
        if not os.path.exists(SAVE_FILE):
            self.notify("No save file found.")
            return

        try:
            with open(SAVE_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)

            for key, value in data["player"].items():
                if hasattr(self.player, key):
                    setattr(self.player, key, value)

            self.mission_index = data.get("mission_index", 0)
            self.day = data.get("day", 1)
            self.hour = data.get("hour", 8)
            self.minute = data.get("minute", 0)

            for saved, car in zip(data.get("cars", []), cars):
                car.owned = saved.get("owned", car.owned)

            self.notify("Game loaded.")
        except (OSError, ValueError, KeyError):
            self.notify("The save file could not be loaded.")

    # ---------- Notifications ----------
    def notify(self, text):
        self.message = text
        self.message_timer = 4

    # ---------- World ----------
    def is_blocked(self, rect):
        return any(rect.colliderect(b) for b in buildings)

    def move_player(self, dx, dy, dt):
        speed = self.player.speed

        if self.in_car and self.active_car:
            speed = self.active_car.speed

        old_x, old_y = self.player.x, self.player.y

        self.player.x += dx * speed * dt
        self.player.y += dy * speed * dt

        self.player.x = clamp(self.player.x, 20, WORLD_W - 20)
        self.player.y = clamp(self.player.y, 20, WORLD_H - 20)

        rect = self.player.rect()

        if self.is_blocked(rect):
            self.player.x = old_x
            self.player.y = old_y

    def update_time(self, dt):
        # Game time advances approximately 1 minute every 2 real seconds.
        self.minute += dt * 30

        while self.minute >= 60:
            self.minute -= 60
            self.hour += 1

        if self.hour >= 24:
            self.hour = 0
            self.day += 1

        if self.player.energy > 0:
            self.player.energy = max(0, self.player.energy - int(dt * 0.7))

    # ---------- Interaction ----------
    def nearest_car(self):
        best = None
        best_dist = 65

        for car in cars:
            d = distance((self.player.x, self.player.y), (car.x, car.y))
            if d < best_dist:
                best_dist = d
                best = car

        return best

    def interact(self):
        # Car interaction
        car = self.nearest_car()

        if car:
            if not car.owned:
                if self.player.money >= car.price:
                    self.player.money -= car.price
                    car.owned = True
                    self.notify(f"You bought the {car.name}.")
                else:
                    self.notify(f"You need ${car.price:,} for this vehicle.")
                return

            if not self.in_car:
                self.in_car = True
                self.active_car = car
                self.player.vehicle_id = cars.index(car)
                car.occupied = True
                self.notify(f"You entered the {car.name}.")
            else:
                self.in_car = False
                car.occupied = False
                self.player.vehicle_id = None
                self.notify("You left the vehicle.")
            return

        # Building interactions
        p = self.player.rect()

        if p.colliderect(BANK):
            amount = self.player.money
            self.player.bank += amount
            self.player.money = 0
            self.notify(f"Deposited ${amount:,} into your bank.")
            return

        if p.colliderect(SHOP):
            price = 50
            if self.player.money >= price:
                self.player.money -= price
                self.player.energy = min(100, self.player.energy + 25)
                self.notify("You bought food and recovered energy.")
            else:
                self.notify("You need $50 for food.")
            return

        if p.colliderect(HOME):
            self.player.energy = 100
            self.player.health = 100
            self.notify("You rested at home.")
            return

        # Mission interaction
        if self.mission_index < len(MISSIONS):
            mission = MISSIONS[self.mission_index]
            target = self.get_target_rect(mission["target"])

            if p.colliderect(target.inflate(40, 40)):
                self.complete_mission()
                return

        self.notify("There is nothing to interact with here.")

    def get_target_rect(self, target):
        return {
            "shop": SHOP,
            "garage": GARAGE,
            "club": CLUB,
            "home": HOME
        }[target]

    def complete_mission(self):
        mission = MISSIONS[self.mission_index]
        self.player.money += mission["reward"]
        self.player.respect += mission["respect"]
        self.player.update_rank()

        self.mission_index += 1

        if self.mission_index >= len(MISSIONS):
            self.notify("Career mission complete! More missions coming soon.")
        else:
            self.notify(
                f"Mission complete! +${mission['reward']:,} and "
                f"+{mission['respect']} respect."
            )

    # ---------- Camera ----------
    def update_camera(self):
        target_x = self.player.x - WIDTH / 2
        target_y = self.player.y - HEIGHT / 2

        self.camera_x = clamp(target_x, 0, WORLD_W - WIDTH)
        self.camera_y = clamp(target_y, 0, WORLD_H - HEIGHT)

    # ---------- Drawing ----------
    def draw_world(self):
        SCREEN.fill(GRASS)

        # Roads
        roads = [
            pygame.Rect(0, 300, WORLD_W, 150),
            pygame.Rect(0, 760, WORLD_W, 150),
            pygame.Rect(0, 1260, WORLD_W, 150),
            pygame.Rect(350, 0, 150, WORLD_H),
            pygame.Rect(1050, 0, 150, WORLD_H),
            pygame.Rect(1500, 0, 150, WORLD_H),
        ]

        for road in roads:
            r = road.move(-self.camera_x, -self.camera_y)
            pygame.draw.rect(SCREEN, ROAD, r)

            # Road markings
            if road.height > road.width:
                x = int(road.centerx - self.camera_x)
                for y in range(-50, HEIGHT + 100, 70):
                    pygame.draw.rect(SCREEN, GOLD, (x - 3, y, 6, 35))
            else:
                y = int(road.centery - self.camera_y)
                for x in range(-50, WIDTH + 100, 70):
                    pygame.draw.rect(SCREEN, GOLD, (x, y - 3, 35, 6))

        # Buildings
        for b in buildings:
            r = b.move(-self.camera_x, -self.camera_y)
            pygame.draw.rect(SCREEN, BUILDING, r, border_radius=4)

            # Windows
            for wx in range(r.left + 20, r.right - 10, 45):
                for wy in range(r.top + 20, r.bottom - 10, 45):
                    pygame.draw.rect(SCREEN, WINDOW, (wx, wy, 20, 24))

        # Important places
        places = [
            (BANK, "BANK", GOLD),
            (SHOP, "SHOP", GREEN),
            (GARAGE, "GARAGE", BLUE),
            (CLUB, "CLUB", (170, 80, 190)),
            (HOME, "HOME", (210, 140, 80)),
        ]

        for rect, label, color in places:
            r = rect.move(-self.camera_x, -self.camera_y)
            pygame.draw.rect(SCREEN, color, r, border_radius=8)
            label_img = FONT.render(label, True, WHITE)
            SCREEN.blit(
                label_img,
                (r.centerx - label_img.get_width() // 2,
                 r.centery - label_img.get_height() // 2)
            )

        # Cars
        for car in cars:
            r = car.rect().move(-self.camera_x, -self.camera_y)

            if car.owned:
                pygame.draw.rect(SCREEN, car.color, r, border_radius=7)
            else:
                pygame.draw.rect(SCREEN, GRAY, r, border_radius=7)

            pygame.draw.rect(
                SCREEN, WINDOW,
                (r.x + 10, r.y + 5, 30, 12)
            )

            if not car.owned:
                draw_text(
                    SCREEN, f"${car.price:,}",
                    SMALL, WHITE, r.x - 5, r.y - 25
                )

        # NPCs
        for npc in npcs:
            npc.draw(SCREEN, (self.camera_x, self.camera_y))

        # Player
        px = int(self.player.x - self.camera_x)
        py = int(self.player.y - self.camera_y)

        if self.in_car:
            pygame.draw.rect(
                SCREEN,
                self.active_car.color,
                (px - 28, py - 16, 56, 32),
                border_radius=8
            )
            pygame.draw.rect(
                SCREEN, WINDOW,
                (px - 17, py - 10, 34, 10)
            )
        else:
            pygame.draw.circle(SCREEN, (235, 190, 150), (px, py - 13), 9)
            pygame.draw.rect(
                SCREEN, (40, 80, 170),
                (px - 10, py - 5, 20, 25),
                border_radius=4
            )

    def draw_hud(self):
        panel = pygame.Surface((WIDTH, 115), pygame.SRCALPHA)
        panel.fill((10, 10, 15, 210))
        SCREEN.blit(panel, (0, 0))

        draw_text(SCREEN, "GOLD VIBES", BIG, GOLD, 20, 14)
        draw_text(SCREEN, f"Cash: ${self.player.money:,}", FONT, WHITE, 235, 18)
        draw_text(SCREEN, f"Bank: ${self.player.bank:,}", FONT, WHITE, 235, 48)
        draw_text(SCREEN, f"Respect: {self.player.respect}", FONT, WHITE, 400, 18)
        draw_text(SCREEN, f"Career Lv: {self.player.rank}", FONT, WHITE, 400, 48)

        draw_text(
            SCREEN,
            f"Health: {self.player.health}",
            SMALL, WHITE, 570, 18
        )
        draw_text(
            SCREEN,
            f"Energy: {self.player.energy}",
            SMALL, WHITE, 570, 48
        )

        # Heat
        draw_text(SCREEN, "Attention:", SMALL, WHITE, 750, 18)
        for i in range(5):
            color = RED if i < self.player.heat else GRAY
            pygame.draw.rect(
                SCREEN, color,
                (850 + i * 24, 20, 18, 18)
            )

        time_text = f"Day {self.day}  {int(self.hour):02d}:{int(self.minute):02d}"
        draw_text(SCREEN, time_text, SMALL, WHITE, 750, 52)

        # Mission
        if self.mission_index < len(MISSIONS):
            m = MISSIONS[self.mission_index]
            draw_text(
                SCREEN,
                f"MISSION: {m['name']}",
                FONT, GOLD, 20, 125
            )
            draw_text(
                SCREEN,
                m["description"],
                SMALL, WHITE, 20, 153
            )
        else:
            draw_text(
                SCREEN,
                "MISSION: Career chapter complete",
                FONT, GOLD, 20, 125
            )

        # Message
        if self.message_timer > 0:
            box = pygame.Rect(WIDTH // 2 - 250, HEIGHT - 75, 500, 45)
            pygame.draw.rect(SCREEN, BLACK, box, border_radius=8)
            text = FONT.render(self.message, True, WHITE)
            SCREEN.blit(
                text,
                (box.centerx - text.get_width() // 2,
                 box.centery - text.get_height() // 2)
            )

    def draw_help(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 190))
        SCREEN.blit(overlay, (0, 0))

        box = pygame.Rect(220, 120, 660, 470)
        pygame.draw.rect(SCREEN, (35, 35, 40), box, border_radius=15)

        draw_text(SCREEN, "GOLD VIBES - CONTROLS", BIG, GOLD, 275, 150)

        controls = [
            "W A S D / Arrow Keys - Move",
            "E - Interact / enter vehicle",
            "P - Pause",
            "C - Career information",
            "H - Help",
            "F5 - Save game",
            "F9 - Load game",
            "ESC - Quit",
            "",
            "Tip: Walk or drive to the highlighted mission location.",
            "Earn money and respect to progress through your career.",
        ]

        y = 220
        for line in controls:
            draw_text(SCREEN, line, FONT, WHITE, 270, y)
            y += 30

    def draw_career(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 170))
        SCREEN.blit(overlay, (0, 0))

        box = pygame.Rect(250, 100, 600, 500)
        pygame.draw.rect(SCREEN, (30, 30, 35), box, border_radius=15)

        draw_text(SCREEN, "CAREER", TITLE, GOLD, 450, 130)
        draw_text(
            SCREEN,
            f"Level {self.player.rank} / 10",
            BIG, WHITE, 450, 210
        )
        draw_text(
            SCREEN,
            f"Respect: {self.player.respect}",
            FONT, WHITE, 450, 260
        )
        draw_text(
            SCREEN,
            f"Cash + Bank: ${self.player.money + self.player.bank:,}",
            FONT, WHITE, 450, 300
        )

        titles = [
            "New Arrival",
            "Street Regular",
            "Connected",
            "Established",
            "Big Player",
            "Business Owner",
            "City Figure",
            "Power Broker",
            "Legend",
            "Gold Vibes Icon",
        ]

        title = titles[self.player.rank - 1]
        draw_text(SCREEN, title, BIG, GOLD, 450, 350)

        draw_text(
            SCREEN,
            "Complete fictional missions to gain respect.",
            SMALL, LIGHT_GRAY, 330, 430
        )
        draw_text(
            SCREEN,
            "Press C to close.",
            SMALL, LIGHT_GRAY, 330, 470
        )

    def draw(self):
        self.draw_world()
        self.draw_hud()

        if self.show_help:
            self.draw_help()

        if self.show_career:
            self.draw_career()

    # ---------- Main update ----------
    def update(self, dt):
        if self.paused or self.show_help or self.show_career:
            return

        keys = pygame.key.get_pressed()

        dx = keys[pygame.K_d] - keys[pygame.K_a]
        dy = keys[pygame.K_s] - keys[pygame.K_w]

        if keys[pygame.K_RIGHT]:
            dx += 1
        if keys[pygame.K_LEFT]:
            dx -= 1
        if keys[pygame.K_DOWN]:
            dy += 1
        if keys[pygame.K_UP]:
            dy -= 1

        if dx != 0 or dy != 0:
            length = (dx * dx + dy * dy) ** 0.5
            dx /= length
            dy /= length

            self.move_player(dx, dy, dt)
            self.update_time(dt)

        for npc in npcs:
            npc.update(dt)

        if self.message_timer > 0:
            self.message_timer -= dt

        # Slowly reduce attention over time.
        if self.player.heat > 0 and random.random() < dt * 0.05:
            self.player.heat -= 1

        self.update_camera()


def main():
    game = Game()
    game.load()

    while game.running:
        dt = CLOCK.tick(FPS) / 1000.0

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                game.running = False

            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    game.running = False

                elif event.key == pygame.K_e and not game.show_help and not game.show_career:
                    game.interact()

                elif event.key == pygame.K_h:
                    game.show_help = not game.show_help
                    game.show_career = False

                elif event.key == pygame.K_c:
                    game.show_career = not game.show_career
                    game.show_help = False

                elif event.key == pygame.K_p:
                    game.paused = not game.paused

                elif event.key == pygame.K_F5:
                    game.save()

                elif event.key == pygame.K_F9:
                    game.load()

        game.update(dt)
        game.draw()

        pygame.display.flip()

    pygame.quit()


if __name__ == "__main__":
    main()

