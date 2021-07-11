# $FreeBSD$

PORTNAME=	palemoon
# UXP_VERSION below *MUST BE UPDATED* according to DISTVERSION (see recorded
# commit of the 'platform' git sub-module and corresponding release in
# UXP/releases; see MASTER_SITES for base URLs).
DISTVERSION=	29.2.1
CATEGORIES=	www
MASTER_SITES=	https://repo.palemoon.org/MoonchildProductions/Pale-Moon/archive/:palemoon \
		https://repo.palemoon.org/MoonchildProductions/UXP/archive/:uxp
DISTFILES=	${DISTVERSION}_Release.tar.gz:palemoon \
		RELBASE_${UXP_VERSION}.tar.gz:uxp
DIST_SUBDIR=	MoonchildProductions

MAINTAINER=	olce.freebsd.ports@certner.fr
COMMENT=	Open-source web browser

LICENSE=	MPL20 MISC
LICENSE_COMB=	multi
LICENSE_NAME_MISC=	Miscellaneous free and open-source licenses
LICENSE_FILE=	${WRKSRC}/LICENSE
LICENSE_FILE_MISC=	${WRKSRC}/platform/toolkit/content/license.html
LICENSE_PERMS_MISC=	dist-mirror pkg-mirror auto-accept

# Not sure about the real status of ARM, but only ARM packages of old versions of
# PM are available in some Linux distros.
ONLY_FOR_ARCHS=	amd64 i386
ONLY_FOR_ARCHS_REASON=Upstream only cares about i386/amd64

# Common deps & build options
BUILD_DEPENDS=	autoconf-2.13:devel/autoconf213 \
		yasm:devel/yasm \
		zip:archivers/zip \

LIB_DEPENDS=	libdbus-1.so:devel/dbus \
		libdbus-glib-1.so:devel/dbus-glib \
		libfontconfig.so:x11-fonts/fontconfig \
		libfreetype.so:print/freetype2

USES=		pkgconfig perl5 gmake gnome xorg desktop-file-utils
USE_PERL5=	build
# XXX
# GTK3 support currently needs gtk20 and gconf2 to pass old-configure (and
# maybe more).
# See also: https://repo.palemoon.org/MoonchildProductions/UXP/issues/1638.
USE_GNOME=	cairo gdkpixbuf2 glib20 pango \
		gtk30 gtk20:build gconf2:build
USE_XORG=	xt x11 xcb xext xrender

# We require GCC (see also GCC_DEFAULT below), but we need to make sure that
# there are no runtime dependencies to libstdc++, i.e., that libc++'s headers
# and libraries are used during compilation and link.
USES+=		compiler:gcc-c++11-lib

# Put this here pending creation of USES=gtar
BUILD_DEPENDS+=	gtar:archivers/gtar
BINARY_ALIAS+=	tar=${LOCALBASE}/bin/gtar

BUILD_DEPENDS+=	tauthon:lang/tauthon
BINARY_ALIAS+=	python=${LOCALBASE}/bin/tauthon \
		python2=${LOCALBASE}/bin/tauthon \
		python2.7=${LOCALBASE}/bin/tauthon

BUNDLE_LIBS=	yes

## Options

# Official branding is enabled in compliance with Pale Moon's redistribution
# license (see https://www.palemoon.org/redist.shtml), point 8b, as explicitly
# confirmed by the owner (Moonchild; see
# https://forum.palemoon.org/viewtopic.php?f=5&t=25625), provided options are
# not modified beyond what is necessary to get a stable build on FreeBSD. So DO
# NOT CHANGE default options without the maintainer's approval.

OPTIONS_SINGLE+=	SOUND
OPTIONS_SINGLE_SOUND=	ALSA PULSEAUDIO
OPTIONS_DEFAULT+=	ALSA

OPTIONS_DEFINE+=	SYNC
OPTIONS_DEFAULT+=	SYNC

OPTIONS_DEFINE+=	SYSTEM_MALLOC

ALSA_BUILD_DEPENDS=	${PREFIX}/include/alsa/asoundlib.h:audio/alsa-lib
ALSA_LIB_DEPENDS=	libasound.so:audio/alsa-lib
PULSEAUDIO_BUILD_DEPENDS=${LOCALBASE}/include/pulse/pulseaudio.h:audio/pulseaudio
PULSEAUDIO_LIB_DEPENDS=	libpulse.so:audio/pulseaudio

SYNC_DESC=		Pale Moon Sync service (see https://www.palemoon.org/sync/)

SYSTEM_MALLOC_DESC=	Use system's jemalloc instead of bundled one

.include <bsd.port.pre.mk>

# We require GCC, and even a specific version of it. 10 is the last version I
# tested and validated, and the last version endorsed upstream. It is also the
# current default in ports, a nice conjunction that cannot always be
# maintained. Version bumps *MUST* be validated by the maintainer (after
# thorough testing, and following upstream's recommendations).
GCC_DEFAULT=	10

## Vars and targets

UXP_VERSION=	20210608

WRKSRC=		${WRKDIR}/pale-moon

DOT_MOZCONFIG=	${WRKSRC}/.mozconfig
PM_BUILD_DIR=	${WRKSRC}/pmbuild

PALEMOON_DESKTOP=${WRKSRC}/palemoon/branding/official/palemoon.desktop

# Extract only -O options
PM_OPTIMIZE=${CFLAGS:M-O*}

.for VAR in PM_BUILD_DIR PM_OPTIMIZE
PM_MOZCONFIG_REINPLACE_ARGS+=-e 's!%%${VAR}%%!${${VAR}}!'
.endfor

# Taken from bsd.gecko.mk. See comment there.
.if defined(DISABLE_MAKE_JOBS) || defined(MAKE_JOBS_UNSAFE)
MAKE_JOBS_NUMBER=	1
.endif
.if defined(MAKE_JOBS_NUMBER)
MOZ_MAKE_FLAGS+=	-j${MAKE_JOBS_NUMBER}
.endif

post-extract:
	${RMDIR} ${WRKSRC}/platform
	${RLN} ${WRKDIR}/uxp ${WRKSRC}/platform

# Prepare '.mozconfig' for 'configure'
pre-configure:
	${CP} ${FILESDIR}/dot.mozconfig ${DOT_MOZCONFIG}
	${REINPLACE_CMD} ${PM_MOZCONFIG_REINPLACE_ARGS} ${DOT_MOZCONFIG}
.if ${PORT_OPTIONS:MALSA}
	${ECHO_CMD} ac_add_options --enable-alsa >> ${DOT_MOZCONFIG}
	${ECHO_CMD} ac_add_options --disable-pulseaudio >> ${DOT_MOZCONFIG}
.endif
.if ! ${PORT_OPTIONS:MSYNC}
	${ECHO_CMD} ac_add_options --disable-sync >> ${DOT_MOZCONFIG}
.endif
.if ${PORT_OPTIONS:MSYSTEM_MALLOC}
	${ECHO_CMD} ac_add_options --disable-jemalloc >> ${DOT_MOZCONFIG}
.else
	${ECHO_CMD} ac_add_options --enable-jemalloc >> ${DOT_MOZCONFIG}
.endif
.if defined(MOZ_MAKE_FLAGS)
	${ECHO_CMD} mk_add_options MOZ_MAKE_FLAGS=${MOZ_MAKE_FLAGS:Q} \
		>> ${DOT_MOZCONFIG}
.endif
# Add build variables
.for VAR in CPP CXXCPP CC CXX CPPFLAGS CFLAGS CXXFLAGS LDFLAGS \
	LD AS AR RANLIB OBJDUMP NM
.if defined(${VAR})
# Remove -rpath options to GCC's directory, we don't need them (no dependencies
# on libs compiled with gfortran, see
# https://wiki.freebsd.org/libgcc%20problem) and we don't want them (triggering
# a link to a different libgcc_s depending on whether GCC is installed at
# runtime).
	${ECHO_CMD} export ${VAR}=\"${${VAR}:N*-rpath*lib/gcc*}\" >> ${DOT_MOZCONFIG}
.endif
.endfor

# Running 'mach configure' separately is "strongly discouraged" (see Mozilla's
# doc on build options configuration). But this does not seem relevant to Pale
# Moon.
do-configure:
	cd ${WRKSRC} && ${SETENV} PATH=${PATH} ./mach configure

do-build:
	cd ${WRKSRC} && ${SETENV} PATH=${PATH} ./mach build
# Going through the package route (the only one documented upstream)
	cd ${WRKSRC} && ${SETENV} PATH=${PATH} ./mach package

do-install:
	${TAR} -C ${STAGEDIR}${PREFIX}/lib \
		-xf ${PM_BUILD_DIR:Q}/dist/palemoon*.tar* \
		palemoon
	${RLN}  ${STAGEDIR}${PREFIX}/lib/palemoon/palemoon \
		${STAGEDIR}${PREFIX}/bin/palemoon
	${INSTALL_DATA} ${PALEMOON_DESKTOP} \
		${STAGEDIR}${PREFIX}/share/applications/palemoon.desktop
	${RLN}  ${STAGEDIR}${PREFIX}/lib/palemoon/browser/icons/mozicon128.png \
		${STAGEDIR}${PREFIX}/share/pixmaps/palemoon.png

.include <bsd.port.mk>
