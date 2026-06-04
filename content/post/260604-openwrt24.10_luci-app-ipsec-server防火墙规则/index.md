---
title: "openwrt24.10_luci-app-ipsec-server防火墙规则"
date: 2026-06-04
draft: false
---

## openwrt24.10_luci-app-ipsec-server防火墙规则

`/etc/init.d/luci-app-ipsec-server`

```sh

#!/bin/sh /etc/rc.common

START=99

CONFIG="luci-app-ipsec-server"
IPSEC_SECRETS_FILE=/etc/ipsec.secrets
IPSEC_CONN_FILE=/etc/ipsec.conf
CHAP_SECRETS=/etc/ppp/chap-secrets
L2TP_PATH=/var/etc/xl2tpd
L2TP_CONTROL_FILE=${L2TP_PATH}/control
L2TP_CONFIG_FILE=${L2TP_PATH}/xl2tpd.conf
L2TP_OPTIONS_FILE=${L2TP_PATH}/options.xl2tpd
L2TP_LOG_FILE=${L2TP_PATH}/xl2tpd.log

vt_clientip=$(uci -q get ${CONFIG}.@service[0].clientip)
l2tp_enabled=$(uci -q get ${CONFIG}.@service[0].l2tp_enable)
l2tp_localip=$(uci -q get ${CONFIG}.@service[0].l2tp_localip)

ipt_flag="IPSec VPN Server"

get_enabled_anonymous_secs() {
	uci -q show "${CONFIG}" | grep "${1}\[.*\.enabled='1'" | cut -d '.' -sf2
}

ipt_rule() {
	# 彻底废弃旧的 iptables 规则函数，避免在新固件中报错
	return 0
}

gen_include() {
	echo '#!/bin/sh' > /var/etc/ipsecvpn.include
	
	cat <<-EOF >> /var/etc/ipsecvpn.include
		# 放行 IPSec IKE 端口 (加单引号强行锁定双引号注释)
		nft 'insert rule inet fw4 input udp dport { 500, 4500 } counter accept comment "IPSec VPN Server"'
		# 放行 ESP 协议 (IPSec 核心数据流)
		nft 'insert rule inet fw4 input ip protocol esp counter accept comment "IPSec VPN Server"'
		# 允许 VPN 客户端网络双向转发
		nft 'insert rule inet fw4 forward ip saddr ${vt_clientip} counter accept comment "IPSec VPN Server"'
		nft 'insert rule inet fw4 forward ip daddr ${vt_clientip} counter accept comment "IPSec VPN Server"'
		# NAT 流量伪装 (MASQUERADE)
		nft 'add rule inet fw4 srcnat ip saddr ${vt_clientip} counter masquerade comment "IPSec VPN Server"'
	EOF

	if [ "${l2tp_enabled}" = 1 ]; then
		cat <<-EOF >> /var/etc/ipsecvpn.include
			# 放行 L2TP 端口
			nft 'insert rule inet fw4 input udp dport 1701 counter accept comment "IPSec VPN Server L2TP"'
			# 允许 L2TP 客户端网络双向转发
			nft 'insert rule inet fw4 forward ip saddr ${l2tp_localip%.*}.0/24 counter accept comment "IPSec VPN Server L2TP"'
			nft 'insert rule inet fw4 forward ip daddr ${l2tp_localip%.*}.0/24 counter accept comment "IPSec VPN Server L2TP"'
			# L2TP NAT 流量伪装
			nft 'add rule inet fw4 srcnat ip saddr ${l2tp_localip%.*}.0/24 counter masquerade comment "IPSec VPN Server L2TP"'
		EOF
	fi

	# 触发 fw4 重新加载，使新生成的 nft 规则立即生效
	/etc/init.d/firewall reload
	return 0
}

start() {
	local vt_enabled=$(uci -q get ${CONFIG}.@service[0].enabled)
	[ "$vt_enabled" = 0 ] && return 1
	
	local vt_gateway="${vt_clientip%.*}.1"
	local vt_secret=$(uci -q get ${CONFIG}.@service[0].secret)
	
	local l2tp_enabled=$(uci -q get ${CONFIG}.@service[0].l2tp_enable)
	[ "${l2tp_enabled}" = 1 ] && {
		touch ${CHAP_SECRETS}
		local vt_remoteip=$(uci -q get ${CONFIG}.@service[0].l2tp_remoteip)
		local ipsec_l2tp_config=$(cat <<-EOF
		#######################################
		# L2TP Connections
		#######################################

		conn L2TP-IKEv1-PSK
		  type=transport
		  keyexchange=ikev1
		  authby=secret
		  leftprotoport=udp/l2tp
		  left=%any
		  right=%any
		  rekey=no
		  forceencaps=yes
		  ike=aes128-sha1-modp2048,aes128-sha1-modp1024,3des-sha1-modp1024,3des-sha1-modp1536
		  esp=aes128-sha1,3des-sha1
		EOF
		)
		
		mkdir -p ${L2TP_PATH}
		cat > ${L2TP_OPTIONS_FILE} <<-EOF
			name "l2tp-server"
			ipcp-accept-local
			ipcp-accept-remote
			ms-dns ${l2tp_localip}
			noccp
			auth
			idle 1800
			mtu 1400
			mru 1400
			lcp-echo-failure 10
			lcp-echo-interval 60
			connect-delay 5000
		EOF
		cat > ${L2TP_CONFIG_FILE} <<-EOF
			[global]
			port = 1701
			;debug avp = yes
			;debug network = yes
			;debug state = yes
			;debug tunnel = yes
			[lns default]
			ip range = ${vt_remoteip}
			local ip = ${l2tp_localip}
			require chap = yes
			refuse pap = yes
			require authentication = no
			name = l2tp-server
			;ppp debug = yes
			pppoptfile = ${L2TP_OPTIONS_FILE}
			length bit = yes
		EOF
		
		local l2tp_users=$(get_enabled_anonymous_secs "@l2tp_users")
		[ -n "${l2tp_users}" ] && {
			for _user in ${l2tp_users}; do
				local u_enabled=$(uci -q get ${CONFIG}.${_user}.enabled)
				[ "${u_enabled}" -eq 1 ] || continue
				
				local u_username=$(uci -q get ${CONFIG}.${_user}.username)
				[ -n "${u_username}" ] || continue
				
				local u_password=$(uci -q get ${CONFIG}.${_user}.password)
				[ -n "${u_password}" ] || continue
				
				local u_ipaddress=$(uci -q get ${CONFIG}.${_user}.ipaddress)
				[ -n "${u_ipaddress}" ] || u_ipaddress="*"
				
				echo "${u_username} l2tp-server ${u_password} ${u_ipaddress}" >> ${CHAP_SECRETS}
			done
		}
		unset user
		
		echo "ip-up-script /usr/share/xl2tpd/ip-up" >> ${L2TP_OPTIONS_FILE}
		echo "ip-down-script /usr/share/xl2tpd/ip-down" >> ${L2TP_OPTIONS_FILE}

		xl2tpd -c ${L2TP_CONFIG_FILE} -C ${L2TP_CONTROL_FILE} -D >${L2TP_LOG_FILE} 2>&1 &
		rm -f "/usr/lib/ipsec/libipsec.so.0"
	}
	
	cat > ${IPSEC_CONN_FILE} <<-EOF
		# ipsec.conf - strongSwan IPsec configuration file

		config setup
		  uniqueids=no
		  charondebug="cfg 2, dmn 2, ike 2, net 0"

		conn %default
		  dpdaction=clear
		  dpddelay=300s
		  rekey=no
		  left=%defaultroute
		  leftfirewall=yes
		  right=%any
		  ikelifetime=60m
		  keylife=20m
		  rekeymargin=3m
		  keyingtries=1
		  auto=add
		  
		#######################################
		# Default non L2TP Connections
		#######################################

		conn Non-L2TP
		  leftsubnet=0.0.0.0/0
		  rightsubnet=${vt_clientip}
		  rightsourceip=${vt_clientip}
		  rightdns=${vt_gateway}
		  ike=aes128-sha1-modp2048,aes128-sha1-modp1024,3des-sha1-modp1024,3des-sha1-modp1536
		  esp=aes128-sha1,3des-sha1

		# Cisco IPSec
		conn IKEv1-PSK-XAuth
		  also=Non-L2TP
		  keyexchange=ikev1
		  leftauth=psk
		  rightauth=psk
		  rightauth2=xauth

		$ipsec_l2tp_config
	EOF

	cat > /etc/ipsec.secrets <<-EOF
	# /etc/ipsec.secrets - strongSwan IPsec secrets file
	: PSK "$vt_secret"
	EOF
	
	local ipsec_users=$(get_enabled_anonymous_secs "@ipsec_users")
	[ -n "${ipsec_users}" ] && {
		for _user in ${ipsec_users}; do
			local u_enabled=$(uci -q get ${CONFIG}.${_user}.enabled)
			[ "${u_enabled}" -eq 1 ] || continue
			
			local u_username=$(uci -q get ${CONFIG}.${_user}.username)
			[ -n "${u_username}" ] || continue
			
			local u_password=$(uci -q get ${CONFIG}.${_user}.password)
			[ -n "${u_password}" ] || continue
			
			echo "${u_username} : XAUTH '${u_password}'" >> ${IPSEC_SECRETS_FILE}
		done
	}
	unset user
	
	ipt_rule add
	
	/usr/lib/ipsec/starter --daemon charon --nofork > /dev/null 2>&1 &
	gen_include
	
	uci -q batch <<-EOF >/dev/null
		set network.ipsec_server.ipaddr="${vt_clientip%.*}.1"
		commit network
	EOF
	ifup ipsec_server > /dev/null 2>&1
}

stop() {
	ifdown ipsec_server > /dev/null 2>&1
	sed -i '/l2tp-server/d' ${CHAP_SECRETS} 2>/dev/null
	top -bn1 | grep "${L2TP_PATH}" | grep -v "grep" | awk '{print $1}' | xargs kill -9 >/dev/null 2>&1
	rm -rf ${L2TP_PATH}
	ps -w | grep "/usr/lib/ipsec" | grep -v "grep" | awk '{print $1}' | xargs kill -9 >/dev/null 2>&1
	ipt_rule del
	rm -rf /var/etc/ipsecvpn.include
	/etc/init.d/firewall reload
	ln -s "libipsec.so.0.0.0" "/usr/lib/ipsec/libipsec.so.0" >/dev/null 2>&1
}

gen_iface_and_firewall() {
	uci -q batch <<-EOF >/dev/null
		delete network.ipsec_server
		set network.ipsec_server=interface
		set network.ipsec_server.device="ipsec0"
		set network.ipsec_server.proto="static"
		set network.ipsec_server.ipaddr="${vt_clientip%.*}.1"
		set network.ipsec_server.netmask="255.255.255.0"
		commit network
		
		delete firewall.ipsecserver
		set firewall.ipsecserver=zone
		set firewall.ipsecserver.name="ipsecserver"
		set firewall.ipsecserver.input="ACCEPT"
		set firewall.ipsecserver.forward="ACCEPT"
		set firewall.ipsecserver.output="ACCEPT"
		set firewall.ipsecserver.network="ipsec_server"
		commit firewall
	EOF
}

if [ -z "$(uci -q get network.ipsec_server)" ] || [ -z "$(uci -q get firewall.ipsecserver)" ]; then
	gen_iface_and_firewall
fi
```
