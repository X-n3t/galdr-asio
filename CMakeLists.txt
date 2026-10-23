cmake_minimum_required(VERSION 3.22)

project(Galdr VERSION 1.1.0)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_subdirectory(JUCE)

# GUI fonts, embedded into the binary. Both are licensed under the SIL OFL
# (see assets/fonts/OFL-*.txt, which must ship alongside any distribution).
juce_add_binary_data(GaldrAssets
    SOURCES
        assets/fonts/UnifrakturMaguntia.ttf
        assets/fonts/IMFellEnglish.ttf)

# PLUGIN_MANUFACTURER_CODE and PLUGIN_CODE must never change after a public
# release, or saved DAW sessions will lose the plugin. The manufacturer code
# was rebranded Admu -> Sgnv once, right after 1.0.0; sessions saved with
# 1.0.0 must re-add the plugin. It is frozen from here on.
juce_add_plugin(Galdr
    COMPANY_NAME "Sognevo"
    BUNDLE_ID com.sognevo.galdr
    IS_SYNTH TRUE
    NEEDS_MIDI_INPUT TRUE
    NEEDS_MIDI_OUTPUT FALSE
    IS_MIDI_EFFECT FALSE
    COPY_PLUGIN_AFTER_BUILD FALSE
    PLUGIN_MANUFACTURER_CODE Sgnv
    PLUGIN_CODE Gldr
    VST3_CATEGORIES Instrument Synth
    FORMATS VST3 AU Standalone
    PRODUCT_NAME "Galdr")

target_sources(Galdr PRIVATE
    src/PluginProcessor.cpp
    src/PluginEditor.cpp)

target_compile_definitions(Galdr PUBLIC
    JUCE_WEB_BROWSER=0
    JUCE_USE_CURL=0
    JUCE_VST3_CAN_REPLACE_VST2=0
    # Disabling the splash screen is permitted when JUCE is used under the (A)GPL.
    JUCE_DISPLAY_SPLASH_SCREEN=0)

# Optional ASIO support for the Windows standalone (off by default).
# JUCE ships without ASIO; the standalone then only offers Windows Audio and
# DirectSound. To enable it, download the ASIO SDK (GPLv3 option, compatible
# with Galdr's licence) and configure with:
#   cmake -B build -DGALDR_ENABLE_ASIO=ON -DASIO_SDK_DIR=C:/path/to/asiosdk
option(GALDR_ENABLE_ASIO "Build with ASIO support (Windows only, needs the ASIO SDK)" OFF)
set(ASIO_SDK_DIR "" CACHE PATH "Folder of the Steinberg ASIO SDK (contains common/ and host/)")
if(GALDR_ENABLE_ASIO)
    if(NOT WIN32)
        message(WARNING "GALDR_ENABLE_ASIO is only meaningful on Windows; ignoring.")
    elseif(NOT EXISTS "${ASIO_SDK_DIR}/common/iasiodrv.h")
        message(FATAL_ERROR
            "GALDR_ENABLE_ASIO=ON but ASIO_SDK_DIR ('${ASIO_SDK_DIR}') does not "
            "contain common/iasiodrv.h. Point ASIO_SDK_DIR at the extracted SDK folder.")
    else()
        target_compile_definitions(Galdr PUBLIC JUCE_ASIO=1)
        target_include_directories(Galdr PUBLIC
            "${ASIO_SDK_DIR}/common"
            "${ASIO_SDK_DIR}/host"
            "${ASIO_SDK_DIR}/host/pc")
    endif()
endif()

target_link_libraries(Galdr
    PRIVATE
        GaldrAssets
        juce::juce_audio_utils
        juce::juce_dsp
    PUBLIC
        juce::juce_recommended_config_flags
        juce::juce_recommended_lto_flags
        juce::juce_recommended_warning_flags)

# Headless DSP tests, run by CI on every platform.
juce_add_console_app(GaldrTests PRODUCT_NAME "GaldrTests")
target_sources(GaldrTests PRIVATE
    tests/Tests.cpp
    src/PluginProcessor.cpp
    src/PluginEditor.cpp)
target_compile_definitions(GaldrTests PRIVATE
    JUCE_WEB_BROWSER=0
    JUCE_USE_CURL=0
    JUCE_DISPLAY_SPLASH_SCREEN=0
    JucePlugin_Name="Galdr")
target_link_libraries(GaldrTests PRIVATE
    GaldrAssets
    juce::juce_audio_utils
    juce::juce_dsp
    juce::juce_recommended_config_flags)

# Offline demo renderer: plays the factory presets to a WAV for the demo video.
# Built on demand, not part of the plugin targets.
juce_add_console_app(GaldrDemoRender PRODUCT_NAME "GaldrDemoRender")
set_target_properties(GaldrDemoRender PROPERTIES EXCLUDE_FROM_ALL TRUE)
target_sources(GaldrDemoRender PRIVATE
    tools/RenderDemo.cpp
    src/PluginProcessor.cpp
    src/PluginEditor.cpp)
target_compile_definitions(GaldrDemoRender PRIVATE
    JUCE_WEB_BROWSER=0
    JUCE_USE_CURL=0
    JUCE_DISPLAY_SPLASH_SCREEN=0
    JucePlugin_Name="Galdr")
target_link_libraries(GaldrDemoRender PRIVATE
    GaldrAssets
    juce::juce_audio_utils
    juce::juce_dsp
    juce::juce_recommended_config_flags)

# CLAP export (MIT-licensed wrapper around the JUCE plugin target)
add_subdirectory(modules/clap-juce-extensions EXCLUDE_FROM_ALL)
clap_juce_extensions_plugin(TARGET Galdr
    CLAP_ID "com.sognevo.galdr"
    CLAP_FEATURES instrument synthesizer)
