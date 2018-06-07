#! / bin / bash
# Versão ligeiramente modificada de https://raw.githubusercontent.com/Nyr/openvpn-install/master/openvpn-install.sh
# Executar com sudo ./openvpn-install.sh e não sudo sh ./openvpn-install.sh como comando de leitura tem problemas com o acionamento externo de scripts
# OpenVPN road warrior installer para Debian, Ubuntu e CentOS

# Este script irá funcionar no Debian, Ubuntu, CentOS e provavelmente outras distribuições
# das mesmas famílias, embora nenhum suporte seja oferecido para elas. Não é
# à prova de balas, mas provavelmente funcionará se você quiser simplesmente configurar uma VPN em
# sua caixa Debian / Ubuntu / CentOS. Foi concebido para ser tão discreto e
# universal quanto possível.


if ["$ USER"! = 'root']; então
	echo "Desculpe, você precisa rodar isso como root"
	Saída
fi


E se [ ! -e / dev / net / tun]; então
	echo "TUN / TAP não está disponível"
	Saída
fi


se grep -q "CentOS release 5" "/ etc / redhat-release"; então
	echo "O CentOS 5 é muito antigo e não é suportado"
	Saída
fi

if [-e / etc / debian_version]; então
	OS = debian
	RCLOCAL = '/ etc / rc.local'
elif [-e / etc / centos-release || -e / etc / redhat-release]; então
	OS = centos
	RCLOCAL = '/ etc / rc.d / rc.local'
	# Necessário para o CentOS 7
	chmod + x /etc/rc.d/rc.local
outro
	echo "Parece que você não está executando este instalador em um sistema Debian, Ubuntu ou CentOS"
	Saída
fi

novo cliente () {
	# Gera o client.ovpn
	cp /usr/share/doc/openvpn*/*ample*/sample-config-files/client.conf ~ / $ 1.ovpn
	sed -i "/ ca ca.crt / d" ~ / $ 1.ovpn
	sed -i "/ cert client.crt / d" ~ / $ 1.ovpn
	sed -i "/ key client.key / d" ~ / $ 1.ovpn
	echo "<ca>" >> ~ / $ 1.ovpn
	cat /etc/openvpn/easy-rsa/2.0/keys/ca.crt >> ~ / $ 1.ovpn
	echo "</ ca>" >> ~ / $ 1.ovpn
	echo "<cert>" >> ~ / $ 1.ovpn
	cat /etc/openvpn/easy-rsa/2.0/keys/$1.crt >> ~ / $ 1.ovpn
	echo "</ cert>" >> ~ / $ 1.ovpn
	echo "<key>" >> ~ / $ 1.ovpn
	cat /etc/openvpn/easy-rsa/2.0/keys/$1.key >> ~ / $ 1.ovpn
	echo "</ key>" >> ~ / $ 1.ovpn
}

geteasyrsa () {
	wget --no-check-certificate -O ~ / easy-rsa.tar.gz https://github.com/OpenVPN/easy-rsa/archive/2.2.2.tar.gz
	tar xzf ~ / easy-rsa.tar.gz -C ~ /
	mkdir -p /etc/openvpn/easy-rsa/2.0/
	cp ~ / easy-rsa-2.2.2 / easy-rsa / 2.0 / * /etc/openvpn/easy-rsa/2.0/
	rm -rf ~ / easy-rsa-2.2.2
	rm -rf ~ / easy-rsa.tar.gz
}


# Tente obter o nosso IP do sistema e faça um fallback para a Internet.
# Eu faço isso para tornar o script compatível com servidores NATed (lowendspirit.com)
# e para evitar um IPv6.
IP = $ (ip addr | grep 'inet' | grep -v inet6 | grep -vE '127 \. [0-9] {1,3} \. [0-9] {1,3} \. [0 -9] {1,3} '| grep -o -E' [0-9] {1,3} \. [0-9] {1,3} \. [0-9] {1,3} \. [0-9] {1,3} '| cabeça -1)
if ["$ IP" = ""]; então
		IP = $ (wget -qO- ipv4.icanhazip.com)
fi


if [-e /etc/openvpn/server.conf]; então
	enquanto :
	Faz
	Claro
		echo "Parece que o OpenVPN já está instalado"
		echo "O que você quer fazer?"
		eco ""
		echo "1) Adicione um certificado para um novo usuário"
		echo "2) Revogar certificado de usuário existente"
		echo "3) Remover o OpenVPN"
		echo "4) Sair"
		eco ""
		read -p "Selecione uma opção [1-4]:" opção
		caso $ option em
			1) 
			eco ""
			echo "Diga-me um nome para o certificado de cliente"
			echo "Por favor, use apenas uma palavra, sem caracteres especiais"
			ler -p "Nome do cliente:" -e -i client CLIENT
			cd /etc/openvpn/easy-rsa/2.0/
			fonte ./vars
			# chave de compilação para o cliente
			export KEY_CN = "$ CLIENTE"
			exportar EASY_RSA = "$ {EASY_RSA: -.}"
			"$ EASY_RSA / pkitool" $ CLIENT
			# Gerar o client.ovpn
			newclient "$ CLIENT"
			eco ""
			echo "Cliente $ CLIENT adicionado, certificados disponíveis em ~ / $ CLIENT.ovpn"
			Saída
			;;
			2)
			eco ""
			echo "Diga-me o nome do cliente existente"
			ler -p "Nome do cliente:" -e -i client CLIENT
			cd /etc/openvpn/easy-rsa/2.0/
			. /etc/openvpn/easy-rsa/2.0/vars
			. /etc/openvpn/easy-rsa/2.0/revoke-full $ CLIENT
			# Se é a primeira vez que revoga um certificado, precisamos adicionar a linha de verificação de URL
			E se ! grep -q "crl-verify" "/etc/openvpn/server.conf"; então
				echo "crl-verify /etc/openvpn/easy-rsa/2.0/keys/crl.pem" >> "/etc/openvpn/server.conf"
				/etc/init.d/openvpn restart
			fi
			eco ""
			echo "Certificado para cliente $ CLIENT revogado"
			Saída
			;;
			3) 
			eco ""
			read -p "Deseja realmente remover o OpenVPN? [y / n]:" -e -in REMOVE
			if ["$ REMOVE" = 'y']; então
				if ["$ OS" = 'debian']; então
					apt-get remove --purge -y openvpn openvpn-lista negra
				outro
					yum remove openvpn -y
				fi
				rm -rf / etc / openvpn
				rm -rf / usr / share / doc / openvpn *
				sed -i '/ iptables -t nat -A POSTROUTING -s 10.8.0.0/d' $ RCLOCAL
				eco ""
				echo "OpenVPN removido!"
			outro
				eco ""
				echo "Remoção cancelada!"
			fi
			Saída
			;;
			4) saída;
		esac
	feito
outro
	Claro
	echo 'Bem-vindo a este rápido OpenVPN "road warrior" installer'
	eco ""
	# Configuração do OpenVPN e criação do primeiro usuário
	echo "Preciso fazer algumas perguntas antes de iniciar a configuração"
	echo "Você pode deixar as opções padrão e pressionar enter se estiver ok com elas"
	eco ""
	echo "Primeiro eu preciso saber o endereço IPv4 da interface de rede que você quer OpenVPN"
	eco "ouvindo".
	ler -p "endereço IP:" -e -i $ IP IP
	eco ""
	echo "Que porta você quer para o OpenVPN?"
	ler -p "Port:" -e -i 1194 PORT
	eco ""
	echo "Deseja que o OpenVPN esteja disponível na porta 53 também?"
	echo "Isso pode ser útil para se conectar em redes restritivas"
	leia -p "Ouça na porta 53 [y / n]:" -e -in ALTPORT
	eco ""
	echo "Deseja habilitar a rede interna para a VPN?"
	echo "Isso pode permitir que clientes VPN se comuniquem entre eles"
	read -p "Permitir rede interna [y / n]:" -e -em INTERNALNETWORK
	eco ""
	echo "Qual DNS você quer usar com a VPN?"
	echo "1) Resolvedores de sistema atuais"
	echo "2) OpenDNS"
	echo "3) nível 3"
	eco "4) NTT"
	echo "5) furacão elétrico"
	eco "6) Yandex"
	leia -p "DNS [1-6]:" -e -i 1 DNS
	eco ""
	echo "Finalmente, diga-me seu nome para o certificado do cliente"
	echo "Por favor, use apenas uma palavra, sem caracteres especiais"
	ler -p "Nome do cliente:" -e -i client CLIENT
	eco ""
	echo "Ok, isso foi tudo que eu precisava. Estamos prontos para configurar seu servidor OpenVPN agora"
	read -n1 -r -p "Pressione qualquer tecla para continuar ..."
	if ["$ OS" = 'debian']; então
		atualização do apt-get
		apt-get instalar o openvpn iptables openssl -y
		cp -R / usr / share / doc / openvpn / exemplos / easy-rsa / / etc / openvpn
		# easy-rsa não está disponível por padrão para o Debian Jessie e mais recente
		E se [ ! -d /etc/openvpn/easy-rsa/2.0/]; então
			geteasyrsa
		fi
	outro
		# Else, a distro é o CentOS
		yum install epel-release -y
		yum instala o openvpn iptables openssl wget -y
		geteasyrsa
	fi
	cd /etc/openvpn/easy-rsa/2.0/
	# Vamos consertar uma coisa primeiro ...
	cp -u -p openssl-1.0.0.cnf openssl.cnf
	# Fuck you NSA - 1024 bits foi o padrão para o Debian Wheezy e mais antigos
	sed -i 's | exportação KEY_SIZE = 1024 | exportação KEY_SIZE = 2048 |' /etc/openvpn/easy-rsa/2.0/vars
	# Crie o PKI
	. /etc/openvpn/easy-rsa/2.0/vars
	. /etc/openvpn/easy-rsa/2.0/clean-all
	# As seguintes linhas são de build-ca. Eu não uso esse script diretamente
	# porque é interativo e não queremos isso. Sim, isso poderia quebrar
	# o script de instalação se o build-ca mudar no futuro.
	exportar EASY_RSA = "$ {EASY_RSA: -.}"
	"$ EASY_RSA / pkitool" --initca $ *
	# Igual à última vez, vamos executar o servidor de chaves de compilação
	exportar EASY_RSA = "$ {EASY_RSA: -.}"
	"$ EASY_RSA / pkitool" - servidor do servidor
	# Agora as chaves do cliente. Precisamos definir KEY_CN ou o pkitool vai chorar
	export KEY_CN = "$ CLIENTE"
	exportar EASY_RSA = "$ {EASY_RSA: -.}"
	"$ EASY_RSA / pkitool" $ CLIENT
	# DH params
	. /etc/openvpn/easy-rsa/2.0/build-dh
	# Vamos configurar o servidor
	cd / usr / share / doc / openvpn * / * amplo * / sample-config-files
	if ["$ OS" = 'debian']; então
		gunzip -d server.conf.gz
	fi
	cp server.conf / etc / openvpn /
	cd /etc/openvpn/easy-rsa/2.0/keys
	cp ca.crt ca.key dh2048.pem server.crt server.key / etc / openvpn
	cd / etc / openvpn /
	# Definir a configuração do servidor
	sed -i 's | dh dh1024.pem | dh dh2048.pem |' server.conf
	sed -i |; push "redirecionamento-gateway def1 bypass-dhcp" | push "redirecionamento-gateway def1 bypass-dhcp" | ' server.conf
	sed -i "s | porta 1194 | porta $ PORT |" server.conf
	# DNS
	caso $ DNS em
		1) 
		# Obtenha os resolvedores do resolv.conf e use-os para o OpenVPN
		grep -v '#' /etc/resolv.conf | grep 'nameserver' | grep -E -o '[0-9] {1,3} \. [0-9] {1,3} \. [0-9] {1,3} \. [0-9] {1, 3} '| enquanto lê a linha; Faz
			sed -i "/; push \" DNS dhcp-option 208.67.220.220 \ "/ a \ push \" dhcp-option DNS $ line \ "" server.conf
		feito
		;;
		2)
		sed -i |; push "dhcp-opção DNS 208.67.222.222" | push "dhcp-option DNS 208.67.222.222" | ' server.conf
		sed -i |; push "dhcp-opção DNS 208.67.220.220" | push "dhcp-option DNS 208.67.220.220" | ' server.conf
		;;
		3) 
		sed -i |; push "dhcp-opção DNS 208.67.222.222" | push "dhcp-option DNS 4.2.2.2" | ' server.conf
		sed -i |; push "dhcp-opção DNS 208.67.220.220" | push "dhcp-option DNS 4.2.2.4" | ' server.conf
		;;
		4) 
		sed -i |; push "dhcp-opção DNS 208.67.222.222" | push "dhcp-option DNS 129.250.35.250" | ' server.conf
		sed -i |; push "dhcp-opção DNS 208.67.220.220" | push "dhcp-option DNS 129.250.35.251" | ' server.conf
		;;
		5) 
		sed -i |; push "dhcp-opção DNS 208.67.222.222" | push "dhcp-option DNS 74.82.42.42" | ' server.conf
		;;
		6) 
		sed -i |; push "dhcp-opção DNS 208.67.222.222" | push "dhcp-option DNS 77.88.8.8" | ' server.conf
		sed -i |; push "DNS dhcp-opção 208.67.220.220" | push "dhcp-option DNS 77.88.8.1" | ' server.conf
		;;
	esac
	# Escute na porta 53 também se o usuário quiser
	if ["$ ALTPORT" = 'y']; então
		sed -i '/ port 1194 / a porta 53' server.conf
	fi
	# Ativar net.ipv4.ip_forward para o sistema
	if ["$ OS" = 'debian']; então
		sed -i 's | # net.ipv4.ip_forward = 1 | net.ipv4.ip_forward = 1 |' /etc/sysctl.conf
	outro
		# CentOS 5 e 6
		sed -i 's | net.ipv4.ip_forward = 0 | net.ipv4.ip_forward = 1 |' /etc/sysctl.conf
		# CentOS 7
		E se ! grep -q "net.ipv4.ip_forward = 1" "/etc/sysctl.conf"; então
			echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
		fi
	fi
	# Evite uma reinicialização desnecessária
	echo 1> / proc / sys / net / ipv4 / ip_forward
	# Set iptables
	if ["$ INTERNALNETWORK" = 'y']; então
		iptables -t nat -A POSTROUTING -s 10.8.0.0/24! -d 10.8.0.0/24 -j SNAT - para $ IP
		sed -i "1 a \ iptables -t nat -A POSTROUTING -s 10.8.0.0/24! -d 10.8.0.0/24 -j SNAT - para $ IP" $ RCLOCAL
	outro
		iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j SNAT - para $ IP
		sed -i "1 a \ iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j SNAT - para $ IP" $ RCLOCAL
	fi
	# E finalmente, reinicie o OpenVPN
	if ["$ OS" = 'debian']; então
		/etc/init.d/openvpn restart
	outro
		# Little hack para verificar se há systemd
		se pidof systemd; então
			systemctl reiniciar openvpn@server.service
			systemctl habilitar openvpn@server.service
		outro
			serviço de reinicialização openvpn
			chkconfig openvpn on
		fi
	fi
	# Tente detectar uma conexão NAT e pergunte sobre isso ao potencial LowEndSpirit
	# Comercial
	EXTERNALIP = $ (wget -qO- ipv4.icanhazip.com)
	if ["$ IP"! = "$ EXTERNALIP"]; então
		eco ""
		echo "Parece que o seu servidor está por trás de um NAT!"
		eco ""
		echo "Se o seu servidor é NATed (LowEndSpirit), eu preciso saber o IP externo"
		echo "Se não for esse o caso, apenas ignore isso e deixe o próximo campo em branco"
		leia -p "IP externo:" -e USEREXTERNALIP
		if ["$ USEREXTERNALIP"! = ""]; então
			IP = $ USEREXTERNALIP
		fi
	fi
	# IP / port definido no client.conf padrão para que possamos adicionar outros usuários
	# sem pedir por eles
	sed -i "s | remoto my-server-1 1194 | remote $ IP $ PORT |" /usr/share/doc/openvpn*/*ample*/sample-config-files/client.conf
	# Gerar o client.ovpn
	newclient "$ CLIENT"
	eco ""
	echo "Concluído!"
	eco ""
	echo "A configuração do seu cliente está disponível em ~ / $ CLIENT.ovpn"
	echo "Se você quiser adicionar mais clientes, basta executar este script outra vez!"
fi
