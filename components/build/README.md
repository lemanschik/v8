# Development Instructions

## Expected Requirements
40+ GB Disk Space is highly Recommended the resulting minimum image is 3.9 gb at present. 
It is not Optimized like squash fs did not run over it but that could maybe improve the build.

## Porting over from gclient
We Do not need to replicate the logic it does we only need to understand the result of its
Operation that allows more efficent rewrites we use this v8 based transperent proxy to 
intercept http(s) request and log the resulting Urls so we can create more efficent
Fetch Based Downloaders for the current release as the Instructions for the download
will not change so often as refactoring in the core project is slow thats why this exists.

```
# capturing traffic from local machine / lan devices behind router.
# 
# uses iptables to redirect all traffic to proxy
# (Your nat tables will be flushed while this is running if you have any)
# must be root (v8_traffic_logger can (should) be run as different user)
#
# source:
#   https://github.com/lemonsqueeze/urldump

usage()
{
    echo "usage:"
    echo "  v8_traffic_logger [options] mitmproxy [args...]"
    echo "  v8_traffic_logger [options] mitmdump  [args...]"
    echo ""
    echo "options:"
    echo "  --local       capture local traffic only"
    echo "  --router      also capture traffic from lan devices behind router"
    echo "                (enabled by default if host is forwarding traffic)"
    echo "  --user name   run v8_traffic_logger command as given user"
    echo "  -d  --debug   debug mode"
    exit 1
}

################################################################################
# Defaults

# Default user to run v8_traffic_logger as (requests from this user won't be logged)
# If you care about security create a non root user.
user=root
#user=v8_traffic_logger

# router ? autodetect
router=""
if grep -q 1 /proc/sys/net/ipv4/ip_forward; then
    router="y"
fi

# Network interfaces when running in router mode (must fill up)
LAN_IF=		# lan interface
NET_IF=		# internet interface

debug=""
#debug=y


################################################################################

die()   {  echo "v8_traffic_logger:" "$@" >&2 ; exit 1;  }
debug() {  if [ -n "$debug" ]; then echo "$@"; fi;  }

while [ "${1#-}" != "$1" ]; do
    opt=$1; shift
    [ "$opt" = "--help" ] && usage
    [ "$opt" = "-h" ]    && usage
    
    if [ "$opt" = "--router" ]; then  router="y"; continue;  fi
    if [ "$opt" = "--local" ];  then  router="";  continue;  fi

    if [ "$opt" = "-d" ]; then      debug="y"; continue; fi
    if [ "$opt" = "--debug" ]; then debug="y"; continue; fi

    if [ "$opt" = "--user" ]; then
	[ $# == 0 ] && usage
	user="$1"; shift; continue
    fi

    die "unknown option: '$opt'"
done

[ $# = 0 ] && usage

# Sanity checks
if [ -n "$router" ]; then
    [ -n "$LAN_IF" ] || die "need to set LAN_IF (lan interface)"
    [ -n "$NET_IF" ] || die "need to set NET_IF (internet interface)"
    # check if nat is enabled
    nat=`iptables-save | grep MASQUERADE`
fi

id "$user" > /dev/null 2>&1 || die "no such user: '$user'"
if [ "$user" = "root" ]; then
    echo "warning: running mitmproxy as root" >&2
fi

type iptables > /dev/null  || die "iptables not found, aborting."
[ `id -u` = 0 ] || die "Must run as root, aborting."

cleanup() {  echo "" >&2;  }
trap cleanup int

# save tables
iptables-save > /tmp/urldump.iptables

# reset tables
for f in filter nat mangle; do
    iptables -t $f -F
    iptables -t $f -X
    iptables -t $f -F
done
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT


# Redirect local http & https requests to proxy
iptables -t nat -A OUTPUT  -p tcp -m owner --uid-owner $user -j ACCEPT
iptables -t nat -A OUTPUT  -p tcp --dport 80 -j REDIRECT --to-port 8080
iptables -t nat -A OUTPUT  -p tcp --dport 443 -j REDIRECT --to-port 8080
# block udp ...
iptables -A OUTPUT  -p udp --dport 80 -j REJECT
iptables -A OUTPUT  -p udp --dport 443 -j REJECT

# Router: Redirect http & https requests from LAN to proxy
if [ -n "$router" ]; then
    iptables -t nat -A PREROUTING -i $LAN_IF -p tcp --dport 80  -j REDIRECT --to-port 8080
    iptables -t nat -A PREROUTING -i $LAN_IF -p tcp --dport 443 -j REDIRECT --to-port 8080
    # ip6tables -t nat -A PREROUTING -i $LAN_IF -p tcp --dport 80  -j REDIRECT --to-port 8080
    # ip6tables -t nat -A PREROUTING -i $LAN_IF -p tcp --dport 443 -j REDIRECT --to-port 8080
    # block udp ...
    iptables -A FORWARD  -p udp --dport 80 -j REJECT
    iptables -A FORWARD  -p udp --dport 443 -j REJECT

    # need this for nat
    if [ -n "$nat" ]; then
	iptables -t nat -A POSTROUTING -o $NET_IF -j MASQUERADE
    fi
    echo "capturing local and lan traffic" >&2
else
    echo "capturing local traffic only" >&2
fi

debug "Starting `basename $1` ..."

# run v8_traffic_logger command
if [ $user = "root" ]; then
                        "$@"
else
    sudo -n -H -u $user "$@"
fi

# restore tables
echo "restoring iptables ..." >&2
iptables-restore /tmp/urldump.iptables
```
