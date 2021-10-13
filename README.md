Guia rápida de instalacion de Gentoo



Esta guía expone cómo instalar Gentoo rápidamente. Gentoo es una distribución que pone su énfasis en las opciones. El handbook oficial de Gentoo sobre la instalación
es un buen recurso para conseguir un sistema Gentoo, pero como el propio manual explica, muchos pasos tienen varias opciones a elegir por el usuario, y se han elegido
las opciones más convenientes para la mayoría de casos. Por lo tanto, en esta guía se proponen otra serie de pasos a seguir, no como una alternativa "mejor" al
handbook, sino eligiendo otras opciones que pueden ser de interés para un usuario nuevo en Gentoo.

Se cubrirá una instalación de Gentoo dirigida a un PC de sobremesa para uso corriente (no para uso en servidor o en arquitecturas más exóticas), por lo que se usará:

- Paquetes binarios
- KDE Plasma
- Systemd
- BTRFS
- Arquitectura x86_64
- UEFI + GPT

Si alguna vez has realizado una instalación de Arch Linux, esta guía debería ser igual o más fácil.


0. Conseguir el medio de instalación
Para la instalación de Gentoo no hace falta usar el Live System de Gentoo, sirve cualquier distro de Linux. En este tutorial usaré el Live System de Ubuntu,
porque ofrece un entorno gráfico para abrir Firefox, usar el ratón en la terminal, etc.

Teniendo la ISO y el pendrive, con el programa [Balena Etcher](https://www.balena.io/etcher/), se flashea la ISO en el pendrive, apagamos el PC y lo iniciamos desde
el pendrive.

Si estamos en una máquina virtual, añadimos el disco ISO y arrancamos desde ahí. También tenemos que asegurarnos
de que el arranque del sistema sea con UEFI y no MBR.

> NOTA: si el sistema tiene Windows ya instalado, hay que desactivar el arranque rápido de Windows.

Verificamos que el sistema verdaderamente haya arrancado desde UEFI:
