# wplace.live-Image-Guide
Um script para o ViolentMonkey que exibe uma imagem de referência sobre o canvas do wplace.live com controles avançados de posicionamento e visualização. Ideal para ajudar na criação de artes pixeladas com precisão!

// ==UserScript==
// @name         wplace.live Image Guide
// @namespace    ViolentMonkey
// @version      3.1
// @description  Mostrador de imagem com ajustes de posição e visualização
// @author       Você
// @match        https://wplace.live/*
// @grant        GM_addStyle
// @grant        GM_getValue
// @grant        GM_setValue
// @require      https://code.jquery.com/jquery-3.6.0.min.js
// ==/UserScript==

(function() {
    'use strict';

    // CONFIGURAÇÃO (COLE SEU BASE64 AQUI)
    const defaultConfig = {
        base64Image: 'data:image/png;base64,COLE_SEU_BASE64_AQUI',
        referenceOpacity: 0.6,
        showReference: true,
        offset: { x: 0, y: 0 },
        uiMinimized: false,
        windowPosition: { x: null, y: null },
        pixelSize: 1
    };

    // Carrega configurações salvas
    let config = { ...defaultConfig, ...GM_getValue('wplaceConfig', {}) };

    // ESTILOS GLOBAIS
    GM_addStyle(`
        #wplace-helper-ui {
            position: fixed;
            bottom: ${config.windowPosition.y !== null ? config.windowPosition.y + 'px' : '20px'};
            right: ${config.windowPosition.x !== null ? config.windowPosition.x + 'px' : '20px'};
            z-index: 9999;
            background: rgba(0,0,0,0.95);
            color: white;
            border-radius: 8px;
            font-family: Arial;
            width: 320px;
            box-shadow: 0 0 20px rgba(0,0,0,0.8);
            transition: all 0.3s ease;
            border: 1px solid #444;
            overflow: hidden;
        }
        
        #wplace-helper-ui.minimized {
            width: 40px;
            height: 40px;
        }
        
        #wplace-header {
            padding: 10px 15px;
            background: rgba(0,0,0,0.7);
            display: flex;
            justify-content: space-between;
            align-items: center;
            cursor: move;
            user-select: none;
        }
        
        #wplace-title {
            font-weight: bold;
            color: #4CAF50;
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
        }
        
        #wplace-window-controls {
            display: flex;
            gap: 8px;
        }
        
        .wplace-window-btn {
            width: 20px;
            height: 20px;
            border-radius: 50%;
            border: none;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 12px;
            padding: 0;
        }
        
        #wplace-minimize-btn {
            background: #FF9800;
            color: white;
        }
        
        #wplace-close-btn {
            background: #f44336;
            color: white;
        }
        
        #wplace-content {
            padding: 15px;
            display: ${config.uiMinimized ? 'none' : 'block'};
        }
        
        #wplace-reference-overlay {
            position: absolute;
            pointer-events: none;
            z-index: 9998;
            opacity: ${config.referenceOpacity};
            image-rendering: pixelated;
            border: 2px dashed rgba(255,255,255,0.3);
        }
        
        .control-group {
            margin-bottom: 12px;
        }
        
        .control-title {
            font-weight: bold;
            margin-bottom: 6px;
            display: block;
        }
    `);

    // VARIÁVEIS DO SISTEMA
    let overlay = null;
    let referenceImage = null;

    // ELEMENTOS DA INTERFACE
    const createUI = () => {
        const ui = $(`
            <div id="wplace-helper-ui" class="${config.uiMinimized ? 'minimized' : ''}">
                <div id="wplace-header">
                    <div id="wplace-title">wplace Image Guide</div>
                    <div id="wplace-window-controls">
                        <button id="wplace-minimize-btn" class="wplace-window-btn">
                            ${config.uiMinimized ? '+' : '-'}
                        </button>
                        <button id="wplace-close-btn" class="wplace-window-btn">×</button>
                    </div>
                </div>
                <div id="wplace-content">
                    <div style="display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-bottom:12px;">
                        <div>
                            <label class="control-title">Opacidade:</label>
                            <input type="range" id="wplace-opacity" min="10" max="90" value="${config.referenceOpacity * 100}" style="width:100%;">
                        </div>
                        <div>
                            <label class="control-title">Tamanho Pixel:</label>
                            <select id="wplace-pixel-size" style="width:100%;padding:5px;">
                                ${[1, 2, 3].map(size => `
                                    <option value="${size}" ${size === config.pixelSize ? 'selected' : ''}>${size}x${size}</option>
                                `).join('')}
                            </select>
                        </div>
                    </div>
                    
                    <div class="control-group" style="margin-bottom:12px;">
                        <label class="control-title">Ajuste de Posição:</label>
                        <div style="display:grid;grid-template-columns:1fr 1fr;gap:8px;">
                            <div>
                                <label style="font-size:12px;">Horizontal (X):</label>
                                <div style="display:flex;gap:5px;">
                                    <button class="pos-adjust" data-axis="x" data-dir="-1" style="flex:1;padding:5px;">←</button>
                                    <button class="pos-adjust" data-axis="x" data-dir="1" style="flex:1;padding:5px;">→</button>
                                </div>
                            </div>
                            <div>
                                <label style="font-size:12px;">Vertical (Y):</label>
                                <div style="display:flex;gap:5px;">
                                    <button class="pos-adjust" data-axis="y" data-dir="-1" style="flex:1;padding:5px;">↑</button>
                                    <button class="pos-adjust" data-axis="y" data-dir="1" style="flex:1;padding:5px;">↓</button>
                                </div>
                            </div>
                        </div>
                        <div style="margin-top:6px;font-size:12px;text-align:center;">
                            Posição atual: X ${config.offset.x}px | Y ${config.offset.y}px
                        </div>
                    </div>
                    
                    <div style="font-size:11px;color:#aaa;text-align:center;">
                        Pressione <strong>Ctrl+Shift+Arrow Keys</strong> para ajustar posição
                    </div>
                </div>
            </div>
        `);

        $('body').append(ui);
        
        // Configura arrastar a janela
        let isDragging = false;
        let offsetX, offsetY;
        
        $('#wplace-header').mousedown(function(e) {
            if (e.target.id !== 'wplace-header' && !$(e.target).closest('#wplace-header').length) return;
            
            isDragging = true;
            offsetX = e.clientX - ui[0].getBoundingClientRect().left;
            offsetY = e.clientY - ui[0].getBoundingClientRect().top;
            ui.css('z-index', '10000');
        });
        
        $(document).mousemove(function(e) {
            if (!isDragging) return;
            
            const x = e.clientX - offsetX;
            const y = e.clientY - offsetY;
            
            ui.css({
                left: x + 'px',
                top: y + 'px',
                right: 'auto',
                bottom: 'auto'
            });
            
            config.windowPosition = { x, y };
        }).mouseup(function() {
            isDragging = false;
            GM_setValue('wplaceConfig', config);
        });
        
        return ui;
    };

    // FUNÇÕES DE CONTROLE DA JANELA
    const toggleMinimize = () => {
        config.uiMinimized = !config.uiMinimized;
        GM_setValue('wplaceConfig', config);
        
        const ui = $('#wplace-helper-ui');
        const content = $('#wplace-content');
        const btn = $('#wplace-minimize-btn');
        
        if (config.uiMinimized) {
            ui.addClass('minimized');
            content.hide();
            btn.text('+');
        } else {
            ui.removeClass('minimized');
            content.show();
            btn.text('-');
        }
    };

    const closeWindow = () => {
        $('#wplace-helper-ui').remove();
        $('#wplace-reference-overlay').remove();
    };

    // CARREGA IMAGEM DE REFERÊNCIA
    const loadReferenceImage = () => {
        if (!config.showReference) return;

        const img = new Image();
        img.onload = function() {
            referenceImage = {
                width: this.width,
                height: this.height
            };
            createOverlay();
            updateOverlayPosition();
        };
        img.onerror = () => {
            console.error('Erro ao carregar imagem de referência');
        };
        img.src = config.base64Image;
    };

    // CRIA OVERLAY DA IMAGEM
    const createOverlay = () => {
        if (overlay) overlay.remove();
        
        overlay = $(`
            <img id="wplace-reference-overlay" 
                 src="${config.base64Image}" 
                 style="opacity: ${config.referenceOpacity};">
        `);
        $('body').append(overlay);
        updateOverlayPosition();
    };

    // ATUALIZA POSIÇÃO DO OVERLAY
    const updateOverlayPosition = () => {
        if (!overlay || !referenceImage) return;
        
        const canvas = document.querySelector('canvas');
        if (!canvas) return;
        
        const rect = canvas.getBoundingClientRect();
        const centerX = rect.width / 2 - (referenceImage.width / 2) + config.offset.x;
        const centerY = rect.height / 2 - (referenceImage.height / 2) + config.offset.y;
        
        overlay.css({
            left: `${rect.left + centerX}px`,
            top: `${rect.top + centerY}px`,
            width: `${referenceImage.width}px`,
            height: `${referenceImage.height}px`
        });
    };

    // AJUSTA POSIÇÃO MANUALMENTE
    const adjustPosition = (axis, direction) => {
        config.offset[axis] += (direction * config.pixelSize);
        GM_setValue('wplaceConfig', config);
        updateOverlayPosition();
        
        $(`.control-group div:contains("Posição atual")`)
            .html(`Posição atual: X ${config.offset.x}px | Y ${config.offset.y}px`);
    };

    // INICIALIZAÇÃO
    const init = () => {
        createUI();
        loadReferenceImage();
        
        // Event Listeners
        $('#wplace-minimize-btn').click(toggleMinimize);
        $('#wplace-close-btn').click(closeWindow);
        
        $('#wplace-opacity').on('input', function() {
            config.referenceOpacity = parseInt($(this).val()) / 100;
            overlay.css('opacity', config.referenceOpacity);
            GM_setValue('wplaceConfig', config);
        });
        
        $('#wplace-pixel-size').change(function() {
            config.pixelSize = parseInt($(this).val());
            GM_setValue('wplaceConfig', config);
        });
        
        $('.pos-adjust').click(function() {
            const axis = $(this).data('axis');
            const dir = $(this).data('dir');
            adjustPosition(axis, dir);
        });
        
        // Teclas de atalho para ajuste de posição
        $(document).keydown(function(e) {
            if (e.ctrlKey && e.shiftKey) {
                if (e.key === 'ArrowUp') {
                    adjustPosition('y', -1);
                    e.preventDefault();
                } else if (e.key === 'ArrowDown') {
                    adjustPosition('y', 1);
                    e.preventDefault();
                } else if (e.key === 'ArrowLeft') {
                    adjustPosition('x', -1);
                    e.preventDefault();
                } else if (e.key === 'ArrowRight') {
                    adjustPosition('x', 1);
                    e.preventDefault();
                }
            }
        });
        
        // Atualiza posição quando o canvas é redimensionado
        const observer = new MutationObserver(updateOverlayPosition);
        observer.observe(document.body, { childList: true, subtree: true });
    };

    // INICIA O SCRIPT
    $(document).ready(function() {
        // Espera o canvas carregar
        const checkCanvas = setInterval(function() {
            if (document.querySelector('canvas')) {
                clearInterval(checkCanvas);
                init();
            }
        }, 500);
    });
})();
