La cpu_0 è l'unica che può avere accesso al file images.h perchè gli altri cores hanno onchip memories piccolissime e non possono leggerlo.

Per velocizzare il programma ho pensato di riorganizzare l'ordine con cui venivano eseguite le operazioni.
La cpu_0, prima ancora di fare il graying, calcola già le coordinate cropX e cropY (da dove farà il taglio). A questo punto fa il graying solo dell'immagine tagliata 31x31 (ho deciso di ritornare alla 31x31 perchè loro ci hanno concesso un errore di 2 pixel e lavorare con immagini più piccole significa risparmiare tempo e memoria) e la scrive nella SHARED ONCHIP memory.
(Giù trovi l'ordine in cui sono scritti i dati nella shared onchip).


A questo punto le altre cpu possono iniziare a lavorare, leggendo dalla shared_onchip una porzione di dati, facendo i conti (individuano le matrici 5x5, le moltiplicano con il pattern, sommano i bit e trovano il massimo).
Ogni cpu scrive il proprio massimo e le coordinate alle quali lo ha trovato.
Questo è il modo in cui le 4 cpu (cpu_1, cpu_2, cpu_3, cpu_4) si sono divise i dati della grayed cropped 36x36 scritta nella shared (ricordati che ci deve essere l'accavallamento di 4 righe per poter leggere bene le matrici 5x5):
- cpu_1 legge dalla riga 0 alla riga 10 (estremi inclusi) --> totale 11 righe;
- cpu_2 legge dalla riga 7 alla riga 17 (estremi inclusi) --> totale 11 righe;
- cpu_3 legge dalla riga 14 alla riga 24 (estremi inclusi) --> totale 11 righe;
- cpu_4 legge dalla riga 21 alla riga 31 (estremi inclusi) --> totale 11 righe;
- cpu_0 lavora dalla riga 28 alla riga 35 (estremi inclusi) --> totale 8 righe;

************************************MODIFICATOO!!!!************************************
La cpu_0 aspetta che tutte le altre trovino il proprio massimo, legge quelli delle altre e calcola il massimo assoluto (e quindi le coordinate effettive del pattern). Aggiorna previousX e previousY (scritte nella shared_onchip) con quelle appena calcolate.
NON è la soluzione ottimale in termini di velocità, perchè mentre si calcola il massimo assoluto si potrebbe procedere con la scrittura in memoria della nuova immagine (da gestire poi con i mutex). Per il momento ho lasciato così perchè non ho avuto tempo di pensare ad una nuova soluzione per la sincronizzazione.
***************************************************************************************

Ecco l'ordine dei dati nella SHARED_ONCHIP a partire dalla riga 0:

previousX              (address: SHARED_ONCHIP_BASE) 	  NON serve più!
previousY              (address: SHARED_ONCHIP_BASE+1)	  NON serve più!
cropX                  (address: SHARED_ONCHIP_BASE+2)	  NON serve più!
cropY                  (address: SHARED_ONCHIP_BASE+3)	  NON serve più!
GRAYED_CROPPED_IMAGE   (from SHARED_ONCHIP_BASE+4)
(36x36)				...
				...
			to SHARED_ONCHIP_BASE+1300)
max1                   (address: SHARED_ONCHIP_BASE+1304) (E' UN INTERO! GLI HO LASCIATO 4 BYTES)
max2                   (address: SHARED_ONCHIP_BASE+1308) (E' UN INTERO! GLI HO LASCIATO 4 BYTES)
max3                   (address: SHARED_ONCHIP_BASE+1312) (E' UN INTERO! GLI HO LASCIATO 4 BYTES)
max4                   (address: SHARED_ONCHIP_BASE+1316) (E' UN INTERO! GLI HO LASCIATO 4 BYTES)
offsetX1               (address: SHARED_ONCHIP_BASE+1333)	
offsetY1               (address: SHARED_ONCHIP_BASE+1334)
offsetX2               (address: SHARED_ONCHIP_BASE+1335)	
offsetY2               (address: SHARED_ONCHIP_BASE+1336)
offsetX3               (address: SHARED_ONCHIP_BASE+1337)	
offsetY3               (address: SHARED_ONCHIP_BASE+1338)
offsetX4               (address: SHARED_ONCHIP_BASE+1339)	
offsetY4               (address: SHARED_ONCHIP_BASE+1340)


Il programma ovviamente non funziona totalmente, però gli ho fatto calcolare le coordinate della prima immagine e le calcola bene (se non ricordo male).
I mutex facevano come al solito casino, perciò per vedere se almeno l'algoritmo era corretto ho commentato tutti i mutex e ho distanziato l'esecuzione delle cpu usando il delay. Praticamente eseguono le cose una dopo l'altra.

Quello che bisogna fare ora è trovare un modo per sincronizzarle.

***************************** NOTE PER VELOCIZZARLE AL PROSSIMO LAB *****************************
1) Tutte le cpu possono calcolare solo l'offset
2) Come detto prima, far calcolare alla cpu4 l'offset con il massimo assoluto
3) Provare ad sostituire moltiplicazioni e divisioni (se possibile) con degli shift

