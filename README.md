# Apocalipse-Red-Devil-Um-V-rus-Mortal
um jogo
import pygame
import random
import math

# --- Configurações do Jogo ---
LARGURA_TELA = 800
ALTURA_TELA = 600
TITULO_JOGO = "Apocalipse Red Devil: Um Vírus Mortal"
FPS = 60

# --- Cores (RGB) ---
PRETO = (0, 0, 0)
BRANCO = (255, 255, 255)
VERMELHO = (255, 0, 0)      # Zumbi / Barra de Saúde (fundo)
VERDE = (0, 255, 0)         # Jogador / Barra de Saúde (preenchimento)
AZUL = (0, 0, 255)
LARANJA = (255, 165, 0)     # Fome
AZUL_CLARO = (0, 191, 255)  # Sede
CINZA = (100, 100, 100)     # Barras de status (fundo)
MARROM = (139, 69, 19)      # Comida
AZUL_MARINHO = (0, 0, 128)  # Água
VERDE_GRAMA = (34, 139, 34) # Cor para a "grama" do mapa
CINZA_ESTRADA = (70, 70, 70) # Cor para a "estrada" do mapa
AMARELO = (255, 255, 0)     # Cor para os projéteis

# --- Configurações do Mapa ---
TAMANHO_TILE = 32
LARGURA_MAPA_TILES = 50
ALTURA_MAPA_TILES = 50
LARGURA_MUNDO_PIXELS = LARGURA_MAPA_TILES * TAMANHO_TILE
ALTURA_MUNDO_PIXELS = ALTURA_MAPA_TILES * TAMANHO_TILE

# --- Inicialização do Pygame ---
pygame.init()
tela = pygame.display.set_mode((LARGURA_TELA, ALTURA_TELA))
pygame.display.set_caption(TITULO_JOGO)
clock = pygame.time.Clock()

# --- Fonte para exibir o texto ---
fonte_pequena = pygame.font.Font(None, 20)
fonte_media = pygame.font.Font(None, 24)
fonte_grande = pygame.font.Font(None, 36)

# --- Classes do Jogo ---

class Camera:
    def __init__(self, largura_mundo, altura_mundo):
        self.camera = pygame.Rect(0, 0, LARGURA_TELA, ALTURA_TELA)
        self.largura_mundo = largura_mundo
        self.altura_mundo = altura_mundo
        self.shake_offset_x = 0
        self.shake_offset_y = 0

    def apply(self, entity):
        return entity.rect.move(self.camera.topleft[0] + self.shake_offset_x,
                                self.camera.topleft[1] + self.shake_offset_y)

    def apply_rect(self, rect):
        return rect.move(self.camera.topleft[0] + self.shake_offset_x,
                        self.camera.topleft[1] + self.shake_offset_y)

    def update(self, target, shake_duracao=0, shake_intensidade=0):
        x = -target.rect.centerx + LARGURA_TELA // 2
        y = -target.rect.centery + ALTURA_TELA // 2

        x = min(0, x)
        y = min(0, y)
        x = max(-(self.largura_mundo - LARGURA_TELA), x)
        y = max(-(self.altura_mundo - ALTURA_TELA), y)

        self.camera = pygame.Rect(x, y, LARGURA_TELA, ALTURA_TELA)

        self.shake_offset_x = 0
        self.shake_offset_y = 0
        if shake_duracao > 0:
            self.shake_offset_x = random.randint(-shake_intensidade, shake_intensidade)
            self.shake_offset_y = random.randint(-shake_intensidade, shake_intensidade)

class Jogador(pygame.sprite.Sprite):
    def __init__(self):
        super().__init__()
        self.image = pygame.Surface((30, 30))
        self.image.fill(VERDE)
        self.rect = self.image.get_rect(center=(LARGURA_MUNDO_PIXELS // 2, ALTURA_MUNDO_PIXELS // 2))
        self.velocidade = 5

        self.fome = 100.0
        self.sede = 100.0
        self.saude = 100.0
        self.ultima_atualizacao_fome_sede = pygame.time.get_ticks()

        self.camerashake_duracao = 0
        self.camerashake_intensidade = 0

        self.inventario = {}
        # Atributos de tiro
        self.munição = 100 # Munição inicial
        self.dano_tiro = 20 # Dano que cada tiro causa
        self.intervalo_tiro = 250 # Milissegundos entre cada tiro (taxa de tiro)
        self.ultimo_tiro = pygame.time.get_ticks() # Tempo do último tiro

    def update(self):
        teclas = pygame.key.get_pressed()
        if teclas[pygame.K_LEFT] or teclas[pygame.K_a]:
            self.rect.x -= self.velocidade
        if teclas[pygame.K_RIGHT] or teclas[pygame.K_d]:
            self.rect.x += self.velocidade
        if teclas[pygame.K_UP] or teclas[pygame.K_w]:
            self.rect.y -= self.velocidade
        if teclas[pygame.K_DOWN] or teclas[pygame.K_s]:
            self.rect.y += self.velocidade

        self.rect.left = max(0, self.rect.left)
        self.rect.right = min(LARGURA_MUNDO_PIXELS, self.rect.right)
        self.rect.top = max(0, self.rect.top)
        self.rect.bottom = min(ALTURA_MUNDO_PIXELS, self.rect.bottom)

        agora = pygame.time.get_ticks()
        if agora - self.ultima_atualizacao_fome_sede > 1500:
            self.fome = max(0.0, self.fome - 0.7)
            self.sede = max(0.0, self.sede - 1.5)
            self.ultima_atualizacao_fome_sede = agora

            if self.fome <= 0:
                self.saude = max(0.0, self.saude - 0.2)
            if self.sede <= 0:
                self.saude = max(0.0, self.saude - 0.5)

        if self.saude <= 0:
            print("GAME OVER! Sua saúde chegou a zero.")
            global rodando
            rodando = False

        if self.camerashake_duracao > 0:
            self.camerashake_duracao -= 1
            if self.camerashake_duracao <= 0:
                self.camerashake_intensidade = 0

    def sofrer_dano(self, quantidade_dano, shake_intensidade=5, shake_duracao=10):
        self.saude = max(0.0, self.saude - quantidade_dano)
        self.camerashake_intensidade = shake_intensidade
        self.camerashake_duracao = shake_duracao
        # print(f"Dano recebido! Saúde: {int(self.saude)}%.")
        if self.saude <= 0:
            print("Você foi sobrepujado pelos zumbis!")
            global rodando
            rodando = False

    def adicionar_item(self, nome_item, quantidade=1):
        self.inventario[nome_item] = self.inventario.get(nome_item, 0) + quantidade
        if nome_item == "Munição": # Se for munição, adiciona ao contador de munição
            self.munição += quantidade * 10 # Cada "Munição" item adiciona 10 balas
            print(f"Munição coletada! Total: {self.munição}")
        # print(f"Item coletado: {nome_item} (x{quantidade}). Inventário: {self.inventario}")

    def usar_item(self, nome_item):
        if nome_item in self.inventario and self.inventario[nome_item] > 0:
            if nome_item == "Comida":
                self.fome = min(100.0, self.fome + 20)
                print("Comida usada! Fome restaurada.")
            elif nome_item == "Agua":
                self.sede = min(100.0, self.sede + 30)
                print("Água usada! Sede restaurada.")
            self.inventario[nome_item] -= 1
            if self.inventario[nome_item] <= 0:
                del self.inventario[nome_item]
            return True
        return False
    
    def atirar(self, projeteis_group, todos_os_sprites_group):
        agora = pygame.time.get_ticks()
        if agora - self.ultimo_tiro > self.intervalo_tiro and self.munição > 0:
            self.ultimo_tiro = agora
            self.munição -= 1
            
            # Pega a posição do mouse na tela
            pos_mouse_x_tela, pos_mouse_y_tela = pygame.mouse.get_pos()

            # Converte a posição do mouse da tela para a posição no mundo
            # Isso é crucial porque o mouse aponta para a tela, mas a bala precisa ir para uma posição no mundo
            # camera.camera.topleft[0] e [1] é o canto superior esquerdo da câmera no mundo
            alvo_x_mundo = pos_mouse_x_tela - camera.camera.topleft[0]
            alvo_y_mundo = pos_mouse_y_tela - camera.camera.topleft[1]

            # Cria um projétil na posição do jogador mirando no alvo (mouse)
            projetil = Projetil(self.rect.centerx, self.rect.centery, alvo_x_mundo, alvo_y_mundo, self.dano_tiro)
            projeteis_group.add(projetil)
            todos_os_sprites_group.add(projetil) # Adiciona ao grupo geral para ser desenhado/atualizado

    def desenhar_barras(self, tela_surface):
        # Desenha barra de Fome
        pygame.draw.rect(tela_surface, CINZA, (10, 10, 100, 15))
        pygame.draw.rect(tela_surface, LARANJA, (10, 10, self.fome, 15))
        texto_fome = fonte_pequena.render(f"Fome: {int(self.fome)}%", True, BRANCO)
        tela_surface.blit(texto_fome, (15, 12))

        # Desenha barra de Sede
        pygame.draw.rect(tela_surface, CINZA, (10, 30, 100, 15))
        pygame.draw.rect(tela_surface, AZUL_CLARO, (10, 30, self.sede, 15))
        texto_sede = fonte_pequena.render(f"Sede: {int(self.sede)}%", True, BRANCO)
        tela_surface.blit(texto_sede, (15, 32))

        # Desenha barra de Saúde
        pygame.draw.rect(tela_surface, VERMELHO, (10, 50, 100, 15))
        pygame.draw.rect(tela_surface, VERDE, (10, 50, self.saude, 15))
        texto_saude = fonte_pequena.render(f"Saúde: {int(self.saude)}%", True, BRANCO)
        tela_surface.blit(texto_saude, (15, 52))

    def desenhar_inventario(self, tela_surface):
        y_offset = ALTURA_TELA - 100
        x_offset = 10
        pygame.draw.rect(tela_surface, CINZA, (x_offset, y_offset, 180, 90), 2)
        titulo_inv = fonte_pequena.render("INVENTÁRIO:", True, BRANCO)
        tela_surface.blit(titulo_inv, (x_offset + 5, y_offset + 5))

        item_y = y_offset + 25
        for item, quantidade in self.inventario.items():
            item_texto = fonte_pequena.render(f"{item}: {quantidade}", True, BRANCO)
            tela_surface.blit(item_texto, (x_offset + 10, item_y))
            item_y += 15
        
        # Desenha a munição separadamente (ou dentro do inventário como um item)
        texto_municao = fonte_media.render(f"Munição: {self.munição}", True, BRANCO)
        tela_surface.blit(texto_municao, (LARGURA_TELA - 120, 10))


class Zumbi(pygame.sprite.Sprite):
    def __init__(self, jogador_ref):
        super().__init__()
        self.image = pygame.Surface((25, 25))
        self.image.fill(VERMELHO)
        self.rect = self.image.get_rect()
        self.rect.x = random.randrange(0, LARGURA_MUNDO_PIXELS - self.rect.width)
        self.rect.y = random.randrange(0, ALTURA_MUNDO_PIXELS - self.rect.height)
        self.velocidade = random.uniform(1.0, 2.5)
        self.jogador_ref = jogador_ref
        self.dano_contato = 5.0
        self.intervalo_dano = 500
        self.ultimo_dano_causado = pygame.time.get_ticks()

        self.vida = 100 # Vida do zumbi
        self.morrendo = False # Flag para animação de morte, se houver

    def update(self):
        if self.vida <= 0:
            # Em uma versão real, aqui você ativaria uma animação de morte
            # Por enquanto, apenas o remove do jogo
            self.kill() # Remove o sprite de todos os grupos em que está

        if not self.morrendo: # Zumbis só se movem se não estiverem morrendo
            dx = self.jogador_ref.rect.centerx - self.rect.centerx
            dy = self.jogador_ref.rect.centery - self.rect.centery
            dist = math.hypot(dx, dy)

            if dist > 0:
                self.rect.x += self.velocidade * (dx / dist)
                self.rect.y += self.velocidade * (dy / dist)

            self.rect.left = max(0, self.rect.left)
            self.rect.right = min(LARGURA_MUNDO_PIXELS, self.rect.right)
            self.rect.top = max(0, self.rect.top)
            self.rect.bottom = min(ALTURA_MUNDO_PIXELS, self.rect.bottom)
    
    def sofrer_dano(self, quantidade_dano):
        self.vida -= quantidade_dano
        # print(f"Zumbi recebeu {quantidade_dano} de dano. Vida restante: {self.vida}")
        if self.vida <= 0:
            self.morrendo = True # Pode ativar uma animação de morte aqui

class Projetil(pygame.sprite.Sprite):
    def __init__(self, x_origem, y_origem, x_alvo, y_alvo, dano):
        super().__init__()
        self.image = pygame.Surface((8, 8)) # Tamanho do projétil
        self.image.fill(AMARELO) # Cor do projétil
        self.rect = self.image.get_rect(center=(x_origem, y_origem))
        self.velocidade = 15 # Velocidade do projétil
        self.dano = dano

        # Calcula a direção do projétil
        dx = x_alvo - x_origem
        dy = y_alvo - y_origem
        dist = math.hypot(dx, dy)
        if dist == 0: # Evita divisão por zero se o jogador atirar no próprio pé
            dist = 1

        self.dir_x = dx / dist
        self.dir_y = dy / dist

    def update(self):
        # Move o projétil na direção calculada
        self.rect.x += self.dir_x * self.velocidade
        self.rect.y += self.dir_y * self.velocidade

        # Remove o projétil se sair da tela (do mundo)
        if (self.rect.x < 0 or self.rect.x > LARGURA_MUNDO_PIXELS or
            self.rect.y < 0 or self.rect.y > ALTURA_MUNDO_PIXELS):
            self.kill() # Remove o projétil de todos os grupos


class Item(pygame.sprite.Sprite):
    def __init__(self, tipo, x, y):
        super().__init__()
        self.tipo = tipo
        self.image = pygame.Surface((20, 20))
        if self.tipo == "Comida":
            self.image.fill(MARROM)
        elif self.tipo == "Agua":
            self.image.fill(AZUL_MARINHO)
        elif self.tipo == "Munição": # Novo tipo de item
            self.image.fill(CINZA) # Cor para a munição
        self.rect = self.image.get_rect()
        self.rect.topleft = (x, y)

# --- Geração do Mapa ---
mapa = []
for y in range(ALTURA_MAPA_TILES):
    linha = []
    for x in range(LARGURA_MAPA_TILES):
        if x % 10 == 0 or y % 10 == 0:
            linha.append(1)
        else:
            linha.append(0)
    mapa.append(linha)


# --- Criação de Sprites e Grupos ---
jogador = Jogador()
todos_os_sprites = pygame.sprite.Group()
itens_coletaveis = pygame.sprite.Group()
zumbis = pygame.sprite.Group()
projeteis = pygame.sprite.Group() # Novo grupo para os projéteis

todos_os_sprites.add(jogador)

# Cria alguns zumbis iniciais
for _ in range(10):
    zumbi = Zumbi(jogador)
    todos_os_sprites.add(zumbi)
    zumbis.add(zumbi)

# Cria alguns itens iniciais (incluindo munição)
for _ in range(8):
    item_tipo = random.choice(["Comida", "Agua", "Munição"]) # Adiciona "Munição"
    item_x = random.randrange(0, LARGURA_MUNDO_PIXELS - TAMANHO_TILE)
    item_y = random.randrange(0, ALTURA_MUNDO_PIXELS - TAMANHO_TILE)
    item = Item(item_tipo, item_x, item_y)
    todos_os_sprites.add(item)
    itens_coletaveis.add(item)


# --- Câmera do Jogo ---
camera = Camera(LARGURA_MUNDO_PIXELS, ALTURA_MUNDO_PIXELS)

# --- Loop Principal do Jogo ---
rodando = True
while rodando:
    # 1. Processamento de Eventos
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            rodando = False
        if event.type == pygame.KEYDOWN:
            if event.key == pygame.K_1:
                jogador.usar_item("Comida")
            if event.key == pygame.K_2:
                jogador.usar_item("Agua")
        if event.type == pygame.MOUSEBUTTONDOWN: # Evento de clique do mouse
            if event.button == 1: # Botão esquerdo do mouse
                jogador.atirar(projeteis, todos_os_sprites) # Chama o método de atirar do jogador

    # 2. Atualização dos Estados do Jogo
    todos_os_sprites.update() # Isso inclui jogador, zumbis, itens e agora projéteis

    # Detecção de Colisão: Jogador vs. Zumbis
    colisoes_zumbi_jogador = pygame.sprite.spritecollide(jogador, zumbis, False)
    agora = pygame.time.get_ticks()
    for zumbi_atingido in colisoes_zumbi_jogador:
        if agora - zumbi_atingido.ultimo_dano_causado > zumbi_atingido.intervalo_dano:
            jogador.sofrer_dano(zumbi_atingido.dano_contato, shake_intensidade=5, shake_duracao=10)
            zumbi_atingido.ultimo_dano_causado = agora

    # Detecção de Colisão: Jogador vs. Itens
    colisoes_itens = pygame.sprite.spritecollide(jogador, itens_coletaveis, True)
    for item_coletado in colisoes_itens:
        jogador.adicionar_item(item_coletado.tipo)
    
    # Detecção de Colisão: Projéteis vs. Zumbis
    # Para cada projétil, verifica colisão com todos os zumbis
    for projetil in projeteis:
        colisoes_projetil_zumbi = pygame.sprite.spritecollide(projetil, zumbis, False) # Não remove o zumbi
        for zumbi_atingido_por_bala in colisoes_projetil_zumbi:
            zumbi_atingido_por_bala.sofrer_dano(projetil.dano) # Zumbi sofre dano do projétil
            projetil.kill() # Remove o projétil ao atingir um zumbi

    camera.update(jogador, jogador.camerashake_duracao, jogador.camerashake_intensidade)

    # 3. Desenho / Renderização
    tela.fill(PRETO)

    # Desenha o Mapa
    for y in range(ALTURA_MAPA_TILES):
        for x in range(LARGURA_MAPA_TILES):
            tile_rect = pygame.Rect(x * TAMANHO_TILE, y * TAMANHO_TILE, TAMANHO_TILE, TAMANHO_TILE)
            screen_tile_rect = camera.apply_rect(tile_rect)
            if screen_tile_rect.colliderect(pygame.Rect(0, 0, LARGURA_TELA, ALTURA_TELA)):
                if mapa[y][x] == 0:
                    pygame.draw.rect(tela, VERDE_GRAMA, screen_tile_rect)
                elif mapa[y][x] == 1:
                    pygame.draw.rect(tela, CINZA_ESTRADA, screen_tile_rect)
    
    # Desenha os sprites
    for sprite in todos_os_sprites:
        tela.blit(sprite.image, camera.apply(sprite))

    jogador.desenhar_barras(tela)
    jogador.desenhar_inventario(tela) # Inventário e munição são desenhados aqui

    pygame.display.flip()

    clock.tick(FPS)

# --- Finaliza o Pygame ---
pygame.quit()
