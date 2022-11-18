# TP C n°3 : les fichiers

Dans ce TP, vous allez manipuler des fichiers en lecture et en écriture non séquentielle, ainsi que les verrous sur fichier.

## Exercice 1 - Lecture/écriture aléatoire

Cet exercice a pour but d'écrire un programme nommé `tri-en-place.c`, qui va trier des entiers long non signées (`unsigned long long`) lus et écrits dans le fichier passé en paramètre.

Le fichier passé en paramètre a une taille de `n * sizeof(unsigned long long)` et devra être lu élément par élément pour le tri. Il faudra pour cela utiliser `fread` et `fwrite` ainsi qu'une ouverture en mode binaire. Vous utiliserez également `ftell` et `fseek` pour vous situer dans le fichier et vous y déplacer.

Vous pouvez également opter pour l'usage de descripteurs de fichiers de type `int`, avec les fonctions `read`, `write` et `lseek`. Pour obtenir la fonctionnalité de `ftell`, `lseek` doit être appelée avec les paramètres `offset` égal à zéro, et `whence` égal à `SEEK_CUR`. La valeur de retour sera alors l'offset actuel de la tête de lecture par rapport au début du fichier.

Vous utiliserez le type de tri que vous souhaitez, même non efficace.

## Exercice 2 - Les processus

Les processus sont des tâches exécutées par le processeur avec un contexte propre à chaque processus. Il existe du point de vue de Linux deux types de tâches :

- les processus, dont il sera question ici, qui sont des tâches disposant de leur propre mémoire (en réalité, c'est un peu plus complexe) et qui sont créés avec la fonction `fork`
- les *threads*, souvent qualifiés de processus légers, qui partagent leur mémoire avec le processus qui les a créés.

Dans cet exercice, nommé `process.c`, vous allez créer des processus et observer leur comportement. La création d'un processus se fait en appelant la fonction `fork`. Cette dernière va cloner le processus qui l'a appelée, et **renvoyer une valeur de retour pour chacun des processus** : le père, i.e. le processus qui a appelé `fork`, ainsi que le fils (le processus créé par l'appel à `fork`). C'est de cette valeur de retour qu'on va déterminer si l'on est dans le père ou le fils :

- Dans le fils, la valeur de retour est égale à 0,
- Dans le père, elle est égale au PID du fils.

Avant de quitter le programme, le père doit attendre (fonction `wait`) chaque fils qu'il a créé.
```c
#include <unistd.h>
#include <sys/wait.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
	if (fork()) { // Parent
		printf("Parent (PID=%d)\n", getpid());
	} else { // Child
		printf("Child (PID=%d)\n", getpid());
		sleep(2);
	}
	wait(NULL);
	printf("[%d] End\n", getpid());
	return 0;
}
```

Remarquez que `fork` duplique l'intégralité du père (sa mémoire et ses instructions comprises). Par conséquent, le fils exécute le bloc d'instructions du `else` ainsi que tout ce qui suit (dont le `printf` avant le `return 0;`). La sortie de ce programme sera donc quelque chose ressemblant à ceci (avec des PID variables) :

```bash
Parent (PID=14366)
Child (PID=14367)
## Ici, une pause de 2 secondes avant l'affichage du reste
[14367] End
[14366] End
```

Pour que le fils retourne avant, et qu'il n'affiche , il faut appeler la fonction `exit` qui prend en paramètre une valeur de retour. Le programme modifié ressemblera donc à ceci :

```c
#include <unistd.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
	if (fork()) { // Parent
		printf("Parent (PID=%d)\n", getpid());
	} else { // Child
		printf("Child (PID=%d)\n", getpid());
		sleep(2);
		exit(0);
	}
	wait(NULL);
	printf("[%d] End\n", getpid());
	return 0;
}
```

qui affichera (on remarquera notamment que le PID affiché avant *End* est bien celui du père) :

```bash
Parent (PID=15021)
Child (PID=15022)
## Ici, une pause de 2 secondes avant l'affichage du reste
[15021] End
```

Pour cet exercice, vous écrirez un programme qui lance deux processus enfants, calculant et affichant pour le premier processus lancé le 1000ème nombre premier, et pour le second, le 98ème nombre premier. Que remarquez vous ?

Vous pourrez vous appuyer sur la fonction suivante pour calculer `i` nombres premiers :

```c
#include <stdint.h> // for uint64_t
#include <math.h> // for sqrt()
void compute_i_primary(int i) {
  int index = 0;
  uint64_t values[i];
  values[index++] = 2;
  values[index++] = 3;
  int n = 5;
  while (index < i) {
    int v;
    for (v=0; v<index && sqrt(n)>=values[v]; ++v) {
      if (n % values[v] == 0) {
        break;
      }
    }
    if (n % values[v] != 0)
      values[index++] = n;
    n += 2;
  }
  // Do something with values, containing the ith first primary numbers
}
```

N'oubliez pas de compiler avec l'option `-lm` pour lier le programme à la bibliothèque mathématique.

## Exercice 3 - Verrous de fichiers

Cet exercice, nommé `filelock.c` nécessite d'écrire un programme qui va verrouiller un fichier dans lequel écrire. Tant que le verrou n'est pas acquis, il est impossible d'écrire dans le fichier. Les verrous sur fichiers avec `flock` fonctionnent comme suit :

- Ouvrir le fichier avec un descripteur numérique (`open`, pas `fopen`)
- Utiliser `flock` sur ce descripteur avec `LOCK_EX` comme opération (second paramètre de la fonction, voir le manuel)
- La fonction retourne (et reprend le flux d'exécution) quand le verrou est acquis.
- Manipuler le fichier
- Relâcher le verrou en appelant à nouveau la fonction `flock` sur le descripteur du fichier, avec cette fois l'opération `LOCK_UN` pour déverrouiller le fichier.

Le programme à écrire prend en paramètre un caractère (qui sera celui qui lui servira à écrire dans le fichier géré par un verrou). Chaque lancement du programme doit itérer 10 fois, et écrire (en le verrouillant, et avec une attente de 1 seconde après verrouillage) 5 lignes de 50 caractères passés en paramètre à chaque itération (chaque itération consiste à verrouiller, écrire, déverrouiller). Vous pourrez lancer les 2 programmes avec la commande suivante :

```bash
./filelock a & ./filelock b
```

pour écrire des lignes de `a` et de `b`. Assurez vous que votre programme permet d'alterner les lignes issues des deux exécutions.

Lors de la manipulation des verrous, il est important de considérer les possibilités de blocage. Par exemple, en manipulant deux fichiers, le programme suivant va se bloquer (à tester en copiant le code dans un source nommé `deadlock.c`) :

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/file.h>

#define FILE1 "test1.txt"
#define FILE2 "test2.txt"

void process1() {
	int fd1 = open(FILE1, O_WRONLY | O_CREAT | 00660);
	int fd2 = open(FILE2, O_WRONLY | O_CREAT | 00660);
	sleep(1);
	printf("[%d] Locking file %s\n", getpid(), FILE1);
	while(flock(fd1, LOCK_EX) != 0);
	printf("[%d] Got lock for %s\n", getpid(), FILE1);
	sleep(1);
	printf("[%d] Locking file %s\n", getpid(), FILE2);
	while(flock(fd2, LOCK_EX) != 0);
	printf("[%d] Got lock for %s\n", getpid(), FILE2);
	sleep(1);
	printf("[%d] Releasing files\n", getpid());
	flock(fd1, LOCK_UN);
	flock(fd2, LOCK_UN);
	close(fd1);
	close(fd2);
}

void process2() {
	int fd1 = open(FILE2, O_WRONLY | O_CREAT | 00660);
	int fd2 = open(FILE1, O_WRONLY | O_CREAT | 00660);
	sleep(1);
	printf("[%d] Locking file %s\n", getpid(), FILE2);
	while(flock(fd1, LOCK_EX) != 0);
	printf("[%d] Got lock for %s\n", getpid(), FILE2);
	sleep(1);
	printf("[%d] Locking file %s\n", getpid(), FILE1);
	while(flock(fd2, LOCK_EX) != 0);
	printf("[%d] Got lock for %s\n", getpid(), FILE1);
	sleep(1);
	printf("[%d] Releasing files\n", getpid());
	flock(fd1, LOCK_UN);
	flock(fd2, LOCK_UN);
	close(fd1);
	close(fd2);
}

int main() {
	if (fork()) {
		process1();
	} else {
		process2();
		exit(0);
	}
	wait(NULL);
	return 0;
}
```

## Bilan

Ce TP a été l'occasion pour vous de découvrir comment se déplacer dans un fichier pour y lire et/ou y écrire n'importe où (donc pas seulement à la fin). Vous y avez également découvert les processus ainsi que le moyen de verrouiller un fichier avant d'y écrire pour permettre à des processus différents de manipuler le même fichier sans effets indésirables. Les fonctions correspondant à ces notions sont les suivantes :

- `fork` pour créer un nouveau processus
- `wait` pour attendre la terminaison des processus enfants
- `flock` pour verrouiller/déverrouiller un fichier
- `ftell` pour indiquer où se situe la tête de lecture/écriture dans un fichier
- `fseek` pour déplacer la tête de lecture dans le fichier
- `lseek` pour remplir les rôles de `fseek` et `ftell` avec un descripteur de fichier au lieu d'un `FILE *`.
