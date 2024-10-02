# Snake
Snake in RiscV Assembly

Оригинальный код был написан мной в рамках дополнительного задания на одном из курсов университета.

Змейка

Номинальный интерфейс:
- Rars
	- Скорость работы max
- Bitmap Display
	- Unit width/height = 32/32
	- Display width/height = 512/512
- Digital Lab Sim
	- 0 - для поворота налево
	- 1 - для поворота направо
- Timer Tool

Если слишком медленно, то изменяя `s9` можно выбрать другое время обновления, сейчас выбранно `1000`, то есть 1 кадр в секунду

```assembly
.eqv CUR 0xFFFF0018 # current time
.eqv NEW 0xFFFF0020 # time for new interrupt

.macro timer()
	lw     t0, CUR
	add   t0, t0, s9
	sw     t0, NEW, t1
.end_macro

.eqv UP 0xff0000
.eqv DOWN 0xff0001
.eqv LEFT 0xff0002
.eqv RIGHT 0xff0003
.eqv APPLE 0x00ff00
.eqv DEBUG 0x000000
.eqv ALLSIZE 0x100 # 16*16
.eqv BASE 0x10010000 # mem adress

.text
	li s0, BASE # Bitmap base
	lui s1, 0xffff0   # MMIO base
	
	li t3, APPLE
	sw t3, 68(s0)
	
	li s2, 8 # y head
	li s3, 8 # x head
	
	slli t2, s2, 6
	slli s4, s3, 2
	add t2, t2, s4
	add t2, t2, s0
	
	li t3, UP
	sw t3, 0(t2)
	
	li s5, 8 # y tail
	li s6, 8 # x tail
	
	li t5, 3 # max len
	li t6, 0 # len
	
	li s9 1000 # timer
	
	li s11 DEBUG
	
	j main
handler:
    timer # generate new timer interupt after 2000 ms.
    uret # return to uepc
    
main:
    la     t0, handler
    csrrw  zero, utvec, t0  # set utvec to the handlers address
    csrrsi zero, ustatus, 0x1 # set interrupt enable bit in ustatus
    csrrsi zero, uie, 0x10 # timer interrupts are enabled (UTIE bit)
    timer # generate new timer interupt after 2000 ms.

main_loop:
	lw t4, 0(t2)

# Check pressed
	li t0, 1
	sb t0, 0x12(s1)
	lb t1, 0x14(s1)
	
	li s4, 0x11
	beq t1, s4, PRESSED_0
	
	li s4, 0x21
	beq t1, s4, PRESSED_1
	
	j PRESSED_NO
# Check pressed


# Transformations
PRESSED_NO: # default
	li s4, UP
	beq t4, s4, NEW_UP
	li s4, DOWN
	beq t4, s4, NEW_DOWN
	li s4, LEFT
	beq t4, s4, NEW_LEFT
	j NEW_RIGHT	
	
PRESSED_0: # pressed left
	li s4, LEFT
	beq t4, s4, NEW_DOWN
	li s4, RIGHT
	beq t4, s4, NEW_UP
	li s4, UP
	beq t4, s4, NEW_LEFT
	j NEW_RIGHT
	
PRESSED_1: # pressed right
	li s4, RIGHT
	beq t4, s4, NEW_DOWN
	li s4, LEFT
	beq t4, s4, NEW_UP
	li s4, DOWN
	beq t4, s4, NEW_LEFT
	j NEW_RIGHT
# Transformations


# Calculate new head position
NEW_UP:
	li t3, UP
	addi s7, zero, -1
	j END_HEAD_IF
NEW_DOWN:
	li t3, DOWN
	addi s7, zero, 1
	j END_HEAD_IF
NEW_LEFT:
	li t3, LEFT
	addi s8, zero, -1
	j END_HEAD_IF
NEW_RIGHT:
	li t3, RIGHT
	addi s8, zero, 1
	j END_HEAD_IF
END_HEAD_IF:
# Calculate new head position


# Fix corner when turning
	slli t2, s2, 6
	slli s4, s3, 2
	add t2, t2, s4
	add t2, t2, s0
	
	sw t3, 0(t2)
# Fix corner when turning	


# Calculate lenght
	addi t6, t6, 1
	bge t5, t6, END_TAIL_IF
	addi t6, t6, -1
# Calculate lenght	
	

# Check tail movement
	slli t2, s5, 6
	slli s4, s6, 2
	add t2, t2, s4
	add t2, t2, s0
	
	lw t4, 0(t2) # Get tail direction
	sw s11, 0(t2) # Cleanup tail
	
	li s4, UP
	beq t4, s4, TAIL_UP
	li s4, DOWN
	beq t4, s4, TAIL_DOWN
	li s4, LEFT
	beq t4, s4, TAIL_LEFT
	j TAIL_RIGHT

TAIL_UP:
	addi s5, s5, -1
	j END_TAIL_IF
TAIL_DOWN:
	addi s5, s5, 1
	j END_TAIL_IF
TAIL_LEFT:
	addi s6, s6, -1
	j END_TAIL_IF
TAIL_RIGHT:
	addi s6, s6, 1
	j END_TAIL_IF
END_TAIL_IF:
# Check tail movement


# Move head 
	add s2, s2, s7
	add s3, s3, s8
	li s7, 0
	li s8, 0
	slli t2, s2, 6
	slli s4, s3, 2
	add t2, t2, s4
	add t2, t2, s0
# Move head 


# Eat apple
	lw t4, 0(t2)
	li s4, APPLE
	beq t4, s4, ENLARGE
	j SKIP
ENLARGE:
	addi t5, t5, 1
	sw zero, 0(t2)
	li t4, 0

GEN_APPLE:
	mv a0, zero
	li a1, ALLSIZE
	li a7, 42
	ecall
	
	slli a0, a0, 2               # make an address by multiplying to 4
    	add  a0, s0, a0
	
	lw s4, 0(a0)
	bgtz s4, GEN_APPLE
	
	li s4, APPLE
	sw s4, 0(a0)
	
	
SKIP:
# Eat apple


# Check G.O.
	li s4, 16
	bge s2, s4, GO
	bge s3, s4, GO
	bltz s2, GO
	bltz s3, GO
	
	bgtz t4, GO
# Check G.O.

	sw t3, 0(t2)
	
	wfi
	j main_loop


GO:
	lui t3, 0xffff0    # MMIO address high half
	li t1, 191
	sb t1, 0x10(t3)   # (0xffff0000+0x10)
	li t2, 189
	sb t2, 0x11(t3)   # (0xffff0001+0x10)
	
```
