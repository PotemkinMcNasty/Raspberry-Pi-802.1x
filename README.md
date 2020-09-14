# Raspberry-Pi-802.1x
Configuration for Raspberry Pi 2020 to use ethernet with 802.1x radius authentication


Configuring a Raspberry Pi (Buster) for wired ethernet radius / 802.1x
EditDelete postReport this postQuote
Mon Sep 14, 2020 4:52 pm

I have a Raspberry Pi 4 running Pi OS based on Debian Buster August 2020 build. It is connected to a network that has the following characteristics:

802.1x secured ethernet
WPA-Enterprise PEAP secured wifi
Radius assigned vlans
DHCP IP v4

I prefer to use the ethernet interface, and needed to configure my Pi to complete 802.1x authentication, then receive an IP address via DHCP. Below is how I did this with help from this forum, and @epoch1970 in particular. Hopefully this will be useful to someone.

I had to amend two files, and create a third:

1. Amend /etc/wpa_supplicant/wpa_supplicant.conf to contain the following

# Where is the control interface located? This is the default path:
# Who can use the WPA frontend? Replace "0" with a group name if you
#  want other users besides root to control it.
#  There should be no need to change this value for a basic configuration:
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev

# IEEE 802.1X works with EAPOL version 2, but the version is defaults 
#   to 1 because of compatibility problems with a number of wireless
#   access points. So we explicitly set it to version 2:
eapol_version=2

# When configuring WPA-Supplicant for use on a wired network, we don’t need to
#   scan for wireless access points. See the wpa-supplicant documentation if
#   you are authenticating through 802.1x on a wireless network:
ap_scan=0

# Set country code
country=GB

# Allow configuration to be updated
update_config=1

# Wired network details.  Note ssid needs to be included but can be anything
network={
        ssid="fakename"
        key_mgmt=IEEE8021X
        eap=PEAP MSCHAPV2
        identity="yourRADIUSidentify"
        password="yourRADIUSpassword"
}
2. Amend /etc/dhcpcd.conf

I added the following to the end of this file:

# Enable fall back to a static IP if DHCP fails:
profile static_eth0
static ip_address=10.0.2.254/24
static routers=10.0.2.1
static domain_name_servers=1.1.1.1 1.0.0.1

# Use env to invoke a 802.1x wired wpa_supplicant dhcpcd-hook in /lib/dhcpcd/dhcpcd-hooks/60-wpa_supplicant_802dot1x
# and fallback to static profile on eth0 if needed
interface eth0
env 802dot1x=1
fallback static_eth0

3. Create /lib/dhcpcd/dhcpcd-hooks/60-wpa_supplicant_802dot1x

# Start, reconfigure and stop wpa_supplicant per wired 802.1x interface.
#
# This is only needed when using wpa_supplicant-2.5 or older, OR
# when wpa_supplicant has not been built with CONFIG_MATCH_IFACE, OR
# wpa_supplicant was launched without the -M flag to activate
# interface matching.

[ -n $802dot1x ] || return 0 

if [ -z "$wpa_supplicant_conf" ]; then
	for x in \
		/etc/wpa_supplicant/wpa_supplicant-"$interface".conf \
		/etc/wpa_supplicant/wpa_supplicant.conf \
		/etc/wpa_supplicant-"$interface".conf \
		/etc/wpa_supplicant.conf \
	; do
		if [ -s "$x" ]; then
			wpa_supplicant_conf="$x"
			break
		fi
	done
fi
: ${wpa_supplicant_conf:=/etc/wpa_supplicant.conf}

wpa_supplicant_ctrldir()
{
	dir=$(key_get_value "[[:space:]]*ctrl_interface=" \
		"$wpa_supplicant_conf")
	dir=$(trim "$dir")
	case "$dir" in
	DIR=*)
		dir=${dir##DIR=}
		dir=${dir%%[[:space:]]GROUP=*}
		dir=$(trim "$dir")
		;;
	esac
	printf %s "$dir"
}

wpa_supplicant_start()
{
	# Pre flight checks
	if [ ! -s "$wpa_supplicant_conf" ]; then
		syslog warn \
			"$wpa_supplicant_conf does not exist"
		syslog warn "not interacting with wpa_supplicant(8)"
		return 1
	fi
	dir=$(wpa_supplicant_ctrldir)
	if [ -z "$dir" ]; then
		syslog warn \
			"ctrl_interface not defined in $wpa_supplicant_conf"
		syslog warn "not interacting with wpa_supplicant(8)"
		return 1
	fi

	wpa_cli -p "$dir" -i "$interface" status >/dev/null 2>&1 && return 0
	syslog info "starting wpa_supplicant"
	wpa_supplicant_driver="${wpa_supplicant_driver:-wired}"
	driver=${wpa_supplicant_driver:+-D}$wpa_supplicant_driver
	err=$(wpa_supplicant -B -c"$wpa_supplicant_conf" -i"$interface" \
	    "$driver" 2>&1)
	errn=$?
	if [ $errn != 0 ]; then
		syslog err "failed to start wpa_supplicant"
		syslog err "$err"
	fi
	return $errn
}

wpa_supplicant_reconfigure()
{
	dir=$(wpa_supplicant_ctrldir)
	[ -z "$dir" ] && return 1
	if ! wpa_cli -p "$dir" -i "$interface" status >/dev/null 2>&1; then
		wpa_supplicant_start
		return $?
	fi
	syslog info "reconfiguring wpa_supplicant"
	err=$(wpa_cli -p "$dir" -i "$interface" reconfigure 2>&1)
	errn=$?
	if [ $errn != 0 ]; then
		syslog err "failed to reconfigure wpa_supplicant"
		syslog err "$err"
	fi
	return $errn
}

wpa_supplicant_stop()
{
	dir=$(wpa_supplicant_ctrldir)
	[ -z "$dir" ] && return 1
	wpa_cli -p "$dir" -i "$interface" status >/dev/null 2>&1 || return 0
	syslog info "stopping wpa_supplicant"
	err=$(wpa_cli -i"$interface" terminate 2>&1)
	errn=$?
	if [ $errn != 0 ]; then
		syslog err "failed to stop wpa_supplicant"
		syslog err "$err"
	fi
	return $errn
}

#if [ "$ifwireless" = "1" ] && \
 if  type wpa_supplicant >/dev/null 2>&1 && \
    type wpa_cli >/dev/null 2>&1
then
	case "$reason" in
	PREINIT)	wpa_supplicant_start;;
	RECONFIGURE)	wpa_supplicant_reconfigure;;
	DEPARTED)	wpa_supplicant_stop;;
	esac
fi
