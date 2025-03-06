import gymnasium as gym
import numpy as np
import pygame

ENABLE_WIND = True
WIND_POWER = 15.0
TURBULENCE_POWER = 0.0
GRAVITY = -10.0
RENDER_MODE = 'human'
RENDER_MODE = None #seleccione esta opção para não visualizar o ambiente (testes mais rápidos)
EPISODES = 1000

env = gym.make("LunarLander-v3", render_mode =RENDER_MODE, 
    continuous=True, gravity=GRAVITY, 
    enable_wind=ENABLE_WIND, wind_power=WIND_POWER, 
    turbulence_power=TURBULENCE_POWER)


def check_successful_landing(observation):
    x = observation[0]  
    vy = observation[3]
    theta = observation[4]
    contact_left = observation[6]
    contact_right = observation[7]

    legs_touching = contact_left == 1 and contact_right == 1

    on_landing_pad = abs(x) <= 0.2

    stable_velocity = vy > -0.2
    stable_orientation = abs(theta) < np.deg2rad(20)
    stable = stable_velocity and stable_orientation
 
    if legs_touching and on_landing_pad and stable:
        print("✅ Aterragem bem sucedida!")
        return True

    print("⚠️ Aterragem falhada!")        
    return False
        
def simulate(steps=1000,seed=None, policy = None):    
    observ, _ = env.reset(seed=seed)
    for step in range(steps):
        action = policy(observ)

        observ, _, term, trunc, _ = env.step(action)

        if term or trunc:
            break

    success = check_successful_landing(observ)
    return step, success



#Perceptions
##TODO: Defina as suas perceções aqui
def perception(observation):
    x = observation[0]
    y = observation[1]
    velx = observation[2]
    vely = observation[3]
    ori = observation[4]
    angvel = observation[5]
    ltouch = observation[6]
    rtouch = observation[7]
    return actions(x,y,velx,vely,ori,angvel,ltouch,rtouch)
    
#Actions
##TODO: Defina as suas ações aqui
def actions(x,y,velx,vely,ori,angvel,ltouch,rtouch):
    action = np.array([0., 0.])
    speedx = 0.2
    
    if ENABLE_WIND == False:
        #orientar para o angulo certo
        if ori < -0.05:
            action[1] -= 0.55
        if ori > 0.05:
            action[1] += 0.55

        #Manter-se no ar e não cair
        if vely < 0:
            action[0] += 0.5
        if y < 0.1 and ltouch and rtouch and x < 0.1 and x > -0.1:
            action[0] = 0
            action[1] = 0

        #Avançar para a direita
        if x < 0 and ori > -0.36 and angvel > -0.16 and velx < 0.19:
            action[1] += 0.051

        #Avançar para a esquerda
        if x > 0 and ori < 0.36 and angvel < 0.16 and velx > -0.19:
            action[1] -= 0.051

    if ENABLE_WIND == True:
        
        #orientar para o angulo certo
        if vely < 0.4 and vely > -0.4:
            if ori < -0.05:
                action[1] -= 0.55
            if ori > 0.05:
                action[1] += 0.55
        elif not(vely < 0.4 and vely > -0.4):
            if ori < -0.05:
                action[1] -= 0.7
            if ori > 0.05:
                action[1] += 0.7

        #Manter-se no ar e não cair
        if y < 0.25 and x < 0.2 and x > -0.2 and ori > -0.3 and ori < 0.3 and velx < 0.4 and velx > -0.4:
            action[0] = 0
            action[1] = 0
        elif vely < 0 and not(x < 0.3 and x > -0.3):
            action[0] += 0.5
            
        elif vely < 0 and x < 0.3 and x > -0.3:
            action[0] += 0.07

        #Avançar para a direita
        if x < 0 and ori > -0.36 and angvel > -0.16 and velx < 0.19:
            action[1] += 0.051

        #hardmode
        if x < -0.45 and angvel > -0.16 and velx < 0:
            action[1] += 1.5

        #Avançar para a esquerda
        if x > 0 and ori < 0.36 and angvel < 0.16 and velx > -0.19:
            action[1] -= 0.051

        #hardmode
        if x > 0.45 and angvel < 0.16 and velx > 0:
            action[1] -= 1.5
        
        #Estacionar à segunda
        if y < 0.1 or (ltouch and rtouch) and not (x < 0.4 and x > -0.4):
            action[0] += 0.3
        
    return action

def reactive_agent(observation):
    ##TODO: Implemente aqui o seu agente reativo
    ##Substitua a linha abaixo pela sua implementação
    
    # print('observação:',observation)
    action = perception(observation)
    # print('\n\nAction:',action)
    
    return action 
    
    
def keyboard_agent(observation):
    action = [0,0] 
    keys = pygame.key.get_pressed()
    
    print('observação:',observation)

    if keys[pygame.K_UP]:  
        action =+ np.array([1,0])
    if keys[pygame.K_LEFT]:  
        action =+ np.array( [0,-1])
    if keys[pygame.K_RIGHT]: 
        action =+ np.array([0,1])

    return action
    

success = 0.0
steps = 0.0
for i in range(EPISODES):
    st, su = simulate(steps=1000000, policy=reactive_agent)

    if su:
        steps += st
    success += su
    
    if su>0:
        print('Média de passos das aterragens bem sucedidas:', steps/success*100)
    print('Taxa de sucesso:', success/(i+1)*100)
    
