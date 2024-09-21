Para explorar as features do teclado V10 é utilizado o software [usevia.app](https://usevia.app) diretamente pelo navegador! E a parte boa é a sua compatibilidade com Linux.

- meu cenário:
  - Fedora release 40 (Forty)
  - Google Chrome
- Basta acessar o link do aplicativo [usevia.app](https://usevia.app) e clicar em **Authorize device**

<figure>
<img src="/src/img/useviaauthorize.png" title="wikilink"
alt="useviaauthorize.png" />
<figcaption aria-hidden="true">useviaauthorize.png</figcaption>
</figure>

> \[!WARNING\]
> Porem ao tentar fazer a autorizacão do dispositivo! Ocorre o erro de permissão de acesso

<figure>
<img src="/src/img/useviaerror.png" title="wikilink" alt="useviaerror.png" />
<figcaption aria-hidden="true">useviaerror.png</figcaption>
</figure>

> \[!note\]
> é preciso criar uma regra `udev` para o driver Linux [hidraw](https://www.kernel.org/doc/Documentation/hid/hidraw.txt) que é utilizado para comunicacão com o teclado.

## Solucão

Primeiro liste as infos do barramento USB pra coletar os dados do device

- [lsusb](https://man7.org/linux/man-pages/man8/lsusb.8.html)
  ou
- [udevadm](https://man7.org/linux/man-pages/man8/udevadm.8.html)

#### Obtendo os dados

- **lsusb**
  Sem rodeios

``` bash
$ lsusb
...
...
Bus 001 Device 005: ID 3434:03a1 Keychron Keychron V10
...
```

> \[!tip\]
> [lsusb](https://man7.org/linux/man-pages/man8/lsusb.8.html#OPTIONS) -D /dev/bus/usb/001/005

- **udevadm**
  Com o [`udevadm`](https://man7.org/linux/man-pages/man8/udevadm.8.html) monitore o sistema [UDEV](https://mirrors.edge.kernel.org/pub/linux/utils/kernel/hotplug/udev/udev.html) [`monitor --property`](https://man7.org/linux/man-pages/man8/udevadm.8.html#OPTIONS)

1.  remova o teclado
2.  inicie o monitoramento
3.  conecte o teclado

``` bash
$ sudo udevadm monitor -up |grep -E "ID_VENDOR_ID|ID_MODEL_ID|DEVNAME"
...
DEVNAME=/dev/bus/usb/001/005
ID_MODEL_ID=03a1
ID_VENDOR_ID=3434
```

Com o path do device em mãos, também podemos obter dados com o comando [`udevadm`](https://man7.org/linux/man-pages/man8/udevadm.8.html) usando a opcão [`info`](https://man7.org/linux/man-pages/man8/udevadm.8.html#OPTIONS)

- nos interessa os campos:
  - ID_MODEL_ID
  - ID_VENDOR_ID

> \[!tip\]
> \$ sudo udevadm info /dev/bus/usb/001/005 \|grep -E "ID_VENDOR_ID\|ID_MODEL_ID"

Sabemos que:
- o barramento é: **001**
- o dispositivo é: **005**
- o ID do vendor Keychron é: **3434**
- o ID do produto Keychron V10 é: **03a1**

## Regra UDEV

Específica para o device [hidraw](https://www.kernel.org/doc/Documentation/hid/hidraw.txt)

- primeiro crie uma variável com o group id do usuário
  - meu usuário é `0xttfx`

  ``` bash
  export USER_GID=`id -g 0xttfx`
  ```
- enfim a regra:

``` bash
$ sudo --preserve-env=USER_GID sh -c 'cat <<EOF > /etc/udev/rules.d/via.rules 
# Keychron_V10
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="3434", ATTRS{idProduct}=="03a1", MODE="0660", GROUP="${USER_GID}", TAG+="uaccess", TAG+="udev-acl"
EOF'
```

Em seguida recarregue as regras UDEV

``` bash
sudo sh -c 'udevadm control --reload-rules && udevadm trigger'
```

Funfando :)
![nice](/src/img/useviaok.png "wikilink")
