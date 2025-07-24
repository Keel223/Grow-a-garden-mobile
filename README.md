# Grow-a-garden-mobile
import React, { useState, useEffect, useRef } from 'react';
import * as THREE from 'three';

const Garden3D = () => {
  const mountRef = useRef(null);
  const sceneRef = useRef(null);
  const rendererRef = useRef(null);
  const cameraRef = useRef(null);
  const plotMeshesRef = useRef([]);
  const plantMeshesRef = useRef([]);
  const frameId = useRef(null);

  // РЎРѕСЃС‚РѕСЏРЅРёРµ РёРіСЂС‹
  const [coins, setCoins] = useState(100);
  const [garden, setGarden] = useState(Array(12).fill(null));
  const [selectedSeed, setSelectedSeed] = useState('tomato');
  const [waterAmount, setWaterAmount] = useState(10);
  const [notifications, setNotifications] = useState([]);
  const [isMobile, setIsMobile] = useState(false);
  const [touchControls, setTouchControls] = useState({
    rotation: 0,
    zoom: 12,
    autoRotate: true
  });
  const [isFirstPerson, setIsFirstPerson] = useState(false);
  const [playerPosition, setPlayerPosition] = useState({ x: 0, z: 0 });
  const [cameraRotation, setCameraRotation] = useState({ horizontal: 0, vertical: 0 });
  const [mobileJoystick, setMobileJoystick] = useState({ x: 0, y: 0, active: false });
  const [gyroEnabled, setGyroEnabled] = useState(false);
  const [gyroSupported, setGyroSupported] = useState(false);
  
  // Р РµС„С‹ РґР»СЏ СѓРїСЂР°РІР»РµРЅРёСЏ
  const keysRef = useRef({});
  const mouseRef = useRef({ isDown: false, lastX: 0, lastY: 0 });
  const joystickRef = useRef(null);
  const gyroRef = useRef({ alpha: 0, beta: 0, gamma: 0, lastAlpha: 0 });

  const seeds = {
    tomato: { name: 'рџЌ… РўРѕРјР°С‚', cost: 10, growTime: 5000, reward: 25, color: 0xff6b6b },
    carrot: { name: 'рџҐ• РњРѕСЂРєРѕРІСЊ', cost: 8, growTime: 4000, reward: 20, color: 0xff9f43 },
    flower: { name: 'рџЊё Р¦РІРµС‚РѕРє', cost: 15, growTime: 3000, reward: 30, color: 0xff6b9d },
    apple: { name: 'рџЌЋ РЇР±Р»РѕРєРѕ', cost: 25, growTime: 8000, reward: 50, color: 0xff4757 },
    corn: { name: 'рџЊЅ РљСѓРєСѓСЂСѓР·Р°', cost: 20, growTime: 6000, reward: 40, color: 0xffa502 },
    strawberry: { name: 'рџЌ“ РљР»СѓР±РЅРёРєР°', cost: 18, growTime: 4500, reward: 35, color: 0xff3838 }
  };

  const addNotification = (message) => {
    const id = Date.now();
    setNotifications(prev => [...prev, { id, message }]);
    setTimeout(() => {
      setNotifications(prev => prev.filter(n => n.id !== id));
    }, 3000);
  };

  // РћРїСЂРµРґРµР»РµРЅРёРµ РјРѕР±РёР»СЊРЅРѕРіРѕ СѓСЃС‚СЂРѕР№СЃС‚РІР° Рё РїРѕРґРґРµСЂР¶РєРё РіРёСЂРѕСЃРєРѕРїР°
  useEffect(() => {
    const checkMobile = () => {
      const isMobileDevice = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent) || 
                            window.innerWidth <= 768;
      setIsMobile(isMobileDevice);
      
      // РџСЂРѕРІРµСЂСЏРµРј РїРѕРґРґРµСЂР¶РєСѓ РіРёСЂРѕСЃРєРѕРїР°
      if (isMobileDevice && 'DeviceOrientationEvent' in window) {
        setGyroSupported(true);
      }
    };
    
    checkMobile();
    window.addEventListener('resize', checkMobile);
    return () => window.removeEventListener('resize', checkMobile);
  }, [isMobile, touchControls]);

  // Р¤СѓРЅРєС†РёСЏ Р·Р°РїСЂРѕСЃР° СЂР°Р·СЂРµС€РµРЅРёСЏ РЅР° РіРёСЂРѕСЃРєРѕРї (РґР»СЏ iOS)
  const requestGyroPermission = async () => {
    if (typeof DeviceOrientationEvent.requestPermission === 'function') {
      try {
        const permission = await DeviceOrientationEvent.requestPermission();
        if (permission === 'granted') {
          setGyroEnabled(true);
          return true;
        }
      } catch (error) {
        console.log('РћС€РёР±РєР° Р·Р°РїСЂРѕСЃР° СЂР°Р·СЂРµС€РµРЅРёСЏ РіРёСЂРѕСЃРєРѕРїР°:', error);
      }
    } else {
      // Р”Р»СЏ Android Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРё РІРєР»СЋС‡Р°РµРј
      setGyroEnabled(true);
      return true;
    }
    return false;
  };

  // РРЅРёС†РёР°Р»РёР·Р°С†РёСЏ 3D СЃС†РµРЅС‹
  useEffect(() => {
    const currentMount = mountRef.current;
    if (!currentMount) return;

    // РЎРѕР·РґР°РЅРёРµ СЃС†РµРЅС‹
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x87CEEB);
    sceneRef.current = scene;

    // РЎРѕР·РґР°РЅРёРµ РєР°РјРµСЂС‹
    const camera = new THREE.PerspectiveCamera(75, currentMount.clientWidth / currentMount.clientHeight, 0.1, 1000);
    camera.position.set(0, 8, 10);
    camera.lookAt(0, 0, 0);
    cameraRef.current = camera;

    const renderer = new THREE.WebGLRenderer({ 
      antialias: !isMobile,
      powerPreference: "high-performance",
      stencil: false,
      depth: true
    });
    renderer.setSize(currentMount.clientWidth, currentMount.clientHeight);
    renderer.shadowMap.enabled = !isMobile;
    if (!isMobile) {
      renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    }
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    currentMount.appendChild(renderer.domElement);
    rendererRef.current = renderer;

    const ambientLight = new THREE.AmbientLight(0x404040, isMobile ? 0.8 : 0.6);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, isMobile ? 0.8 : 1);
    directionalLight.position.set(5, 10, 5);
    if (!isMobile) {
      directionalLight.castShadow = true;
      directionalLight.shadow.mapSize.width = 1024;
      directionalLight.shadow.mapSize.height = 1024;
      directionalLight.shadow.camera.near = 0.5;
      directionalLight.shadow.camera.far = 50;
    }
    scene.add(directionalLight);

    // РЎРѕР·РґР°РЅРёРµ Р·РµРјР»Рё
    const groundGeometry = new THREE.PlaneGeometry(20, 20);
    const groundMaterial = new THREE.MeshLambertMaterial({ color: 0x228B22 });
    const ground = new THREE.Mesh(groundGeometry, groundMaterial);
    ground.rotation.x = -Math.PI / 2;
    if (!isMobile) {
      ground.receiveShadow = true;
    }
    scene.add(ground);

    // РЎРѕР·РґР°РЅРёРµ РіСЂСЏРґРѕРє (3x4 СЃРµС‚РєР°)
    const plotMeshes = [];
    for (let i = 0; i < 12; i++) {
      const row = Math.floor(i / 3);
      const col = i % 3;
      
      const plotGeometry = new THREE.BoxGeometry(1.5, 0.2, 1.5);
      const plotMaterial = new THREE.MeshLambertMaterial({ color: 0x8B4513 });
      const plotMesh = new THREE.Mesh(plotGeometry, plotMaterial);
      
      plotMesh.position.set((col - 1) * 2, 0.1, (row - 1.5) * 2);
      if (!isMobile) {
        plotMesh.receiveShadow = true;
      }
      plotMesh.userData = { plotIndex: i };
      
      scene.add(plotMesh);
      plotMeshes.push(plotMesh);
    }
    plotMeshesRef.current = plotMeshes;

    // РЈРїСЂР°РІР»РµРЅРёРµ РєР°РјРµСЂРѕР№ РґР»СЏ РґРµСЃРєС‚РѕРїР° Рё РјРѕР±Р°Р№Р»Р°
    let mouseX = 0;
    let mouseY = 0;
    let lastTouchX = 0;
    let lastTouchY = 0;
    let touchStartDistance = 0;

    const onMouseMove = (event) => {
      if (!isMobile) {
        if (isFirstPerson && mouseRef.current.isDown) {
          // РЈРїСЂР°РІР»РµРЅРёРµ РєР°РјРµСЂРѕР№ РѕС‚ РїРµСЂРІРѕРіРѕ Р»РёС†Р°
          const deltaX = event.clientX - mouseRef.current.lastX;
          const deltaY = event.clientY - mouseRef.current.lastY;
          
          setCameraRotation(prev => ({
            horizontal: prev.horizontal - deltaX * 0.002,
            vertical: Math.max(-Math.PI/3, Math.min(Math.PI/3, prev.vertical - deltaY * 0.002))
          }));
          
          mouseRef.current.lastX = event.clientX;
          mouseRef.current.lastY = event.clientY;
        } else {
          mouseX = (event.clientX / window.innerWidth) * 2 - 1;
          mouseY = -(event.clientY / window.innerHeight) * 2 + 1;
        }
      }
    };

    const onMouseDown = (event) => {
      if (isFirstPerson && !isMobile) {
        mouseRef.current.isDown = true;
        mouseRef.current.lastX = event.clientX;
        mouseRef.current.lastY = event.clientY;
      }
    };

    const onMouseUp = (event) => {
      mouseRef.current.isDown = false;
    };

    // РЎРµРЅСЃРѕСЂРЅРѕРµ СѓРїСЂР°РІР»РµРЅРёРµ
    const onTouchStart = (event) => {
      event.preventDefault();
      setTouchControls(prev => ({ ...prev, autoRotate: false }));
      
      if (event.touches.length === 1) {
        lastTouchX = event.touches[0].clientX;
        lastTouchY = event.touches[0].clientY;
      } else if (event.touches.length === 2) {
        const dx = event.touches[0].clientX - event.touches[1].clientX;
        const dy = event.touches[0].clientY - event.touches[1].clientY;
        touchStartDistance = Math.sqrt(dx * dx + dy * dy);
      }
    };

    const onTouchMove = (event) => {
      event.preventDefault();
      
      if (event.touches.length === 1) {
        if (isFirstPerson) {
          // РџРѕРІРѕСЂРѕС‚ РєР°РјРµСЂС‹ РѕС‚ РїРµСЂРІРѕРіРѕ Р»РёС†Р° РЅР° РјРѕР±Р°Р№Р»Рµ
          const deltaX = event.touches[0].clientX - lastTouchX;
          const deltaY = event.touches[0].clientY - lastTouchY;
          
          setCameraRotation(prev => ({
            horizontal: prev.horizontal - deltaX * 0.003,
            vertical: Math.max(-Math.PI/3, Math.min(Math.PI/3, prev.vertical - deltaY * 0.003))
          }));
        } else {
          // РџРѕРІРѕСЂРѕС‚ РєР°РјРµСЂС‹ РѕРґРЅРёРј РїР°Р»СЊС†РµРј РІ РѕР±С‹С‡РЅРѕРј СЂРµР¶РёРјРµ
          const deltaX = event.touches[0].clientX - lastTouchX;
          setTouchControls(prev => ({
            ...prev,
            rotation: prev.rotation + deltaX * 0.01
          }));
        }
        lastTouchX = event.touches[0].clientX;
        lastTouchY = event.touches[0].clientY;
      } else if (event.touches.length === 2 && !isFirstPerson) {
        // Р—СѓРј РґРІСѓРјСЏ РїР°Р»СЊС†Р°РјРё (С‚РѕР»СЊРєРѕ РІ РѕР±С‹С‡РЅРѕРј СЂРµР¶РёРјРµ)
        const dx = event.touches[0].clientX - event.touches[1].clientX;
        const dy = event.touches[0].clientY - event.touches[1].clientY;
        const distance = Math.sqrt(dx * dx + dy * dy);
        const scale = distance / touchStartDistance;
        
        setTouchControls(prev => ({
          ...prev,
          zoom: Math.max(6, Math.min(20, prev.zoom / scale))
        }));
        touchStartDistance = distance;
      }
    };

    const onTouchEnd = (event) => {
      event.preventDefault();
      if (event.touches.length === 0) {
        // Р’РѕР·РѕР±РЅРѕРІР»СЏРµРј Р°РІС‚РѕРїРѕРІРѕСЂРѕС‚ С‡РµСЂРµР· 3 СЃРµРєСѓРЅРґС‹ Р±РµР·РґРµР№СЃС‚РІРёСЏ
        setTimeout(() => {
          setTouchControls(prev => ({ ...prev, autoRotate: true }));
        }, 3000);
      }
    };

    // РљР»РёРєРё РјС‹С€СЊСЋ Рё РєР°СЃР°РЅРёСЏ

    const onMouseClick = (event) => {
      if (!isMobile) {
        handleInteraction(event.clientX, event.clientY);
      }
    };

    // РћР±СЂР°Р±РѕС‚РєР° РєР°СЃР°РЅРёР№ (С‚Р°Рї)
    let touchStartTime = 0;
    let touchMoved = false;
    
    const onTouchStartForClick = (event) => {
      touchStartTime = Date.now();
      touchMoved = false;
    };
    
    const onTouchMoveForClick = (event) => {
      touchMoved = true;
    };
    
    const onTouchEndForClick = (event) => {
      const touchDuration = Date.now() - touchStartTime;
      if (!touchMoved && touchDuration < 500 && event.changedTouches.length === 1) {
        const touch = event.changedTouches[0];
        handleInteraction(touch.clientX, touch.clientY);
      }
    };

    // РћР±СЂР°Р±РѕС‚С‡РёРєРё РєР»Р°РІРёР°С‚СѓСЂС‹ РґР»СЏ С…РѕРґСЊР±С‹
    const onKeyDown = (event) => {
      keysRef.current[event.code] = true;
      if (event.code === 'KeyF') {
        // F - РїРµСЂРµРєР»СЋС‡РµРЅРёРµ СЂРµР¶РёРјР° РѕС‚ РїРµСЂРІРѕРіРѕ Р»РёС†Р°
        setIsFirstPerson(prev => !prev);
      }
      if (event.code === 'Escape' && isFirstPerson) {
        setIsFirstPerson(false);
      }
    };

    const onKeyUp = (event) => {
      keysRef.current[event.code] = false;
    };

    // Р”РѕР±Р°РІР»СЏРµРј РѕР±СЂР°Р±РѕС‚С‡РёРєРё СЃРѕР±С‹С‚РёР№
    renderer.domElement.addEventListener('mousemove', onMouseMove);
    renderer.domElement.addEventListener('mousedown', onMouseDown);
    renderer.domElement.addEventListener('mouseup', onMouseUp);
    renderer.domElement.addEventListener('click', onMouseClick);
    document.addEventListener('keydown', onKeyDown);
    document.addEventListener('keyup', onKeyUp);
    
    // РњРѕР±РёР»СЊРЅС‹Рµ РѕР±СЂР°Р±РѕС‚С‡РёРєРё
    if (isMobile) {
      renderer.domElement.addEventListener('touchstart', onTouchStart, { passive: false });
      renderer.domElement.addEventListener('touchmove', onTouchMove, { passive: false });
      renderer.domElement.addEventListener('touchend', onTouchEnd, { passive: false });
      
      // РћС‚РґРµР»СЊРЅС‹Рµ РѕР±СЂР°Р±РѕС‚С‡РёРєРё РґР»СЏ РєР»РёРєРѕРІ
      renderer.domElement.addEventListener('touchstart', onTouchStartForClick);
      renderer.domElement.addEventListener('touchmove', onTouchMoveForClick);
      renderer.domElement.addEventListener('touchend', onTouchEndForClick);
    }

    // Р¤СѓРЅРєС†РёСЏ РѕР±РЅРѕРІР»РµРЅРёСЏ РїРѕР·РёС†РёРё РёРіСЂРѕРєР°
    const updatePlayerMovement = () => {
      if (!isFirstPerson) return;
      
      const moveSpeed = 0.15; // РЈРІРµР»РёС‡РёР»Рё Р±Р°Р·РѕРІСѓСЋ СЃРєРѕСЂРѕСЃС‚СЊ
      let deltaX = 0;
      let deltaZ = 0;
      
      if (keysRef.current['KeyW'] || keysRef.current['ArrowUp'] || mobileJoystick.y < -0.3) {
        deltaZ -= moveSpeed;
      }
      if (keysRef.current['KeyS'] || keysRef.current['ArrowDown'] || mobileJoystick.y > 0.3) {
        deltaZ += moveSpeed;
      }
      if (keysRef.current['KeyA'] || keysRef.current['ArrowLeft'] || mobileJoystick.x < -0.3) {
        deltaX -= moveSpeed;
      }
      if (keysRef.current['KeyD'] || keysRef.current['ArrowRight'] || mobileJoystick.x > 0.3) {
        deltaX += moveSpeed;
      }
      
      if (isMobile && mobileJoystick.active) {
        const joystickMultiplier = 1.8;
        
        const joystickDeltaX = mobileJoystick.x * moveSpeed * joystickMultiplier;
        const joystickDeltaZ = -mobileJoystick.y * moveSpeed * joystickMultiplier;
        
        const cos = Math.cos(cameraRotation.horizontal);
        const sin = Math.sin(cameraRotation.horizontal);
        
        const rotatedDeltaX = joystickDeltaX * cos - joystickDeltaZ * sin;
        const rotatedDeltaZ = joystickDeltaX * sin + joystickDeltaZ * cos;
        
        deltaX += rotatedDeltaX;
        deltaZ += rotatedDeltaZ;
      }
      
      if (deltaX !== 0 || deltaZ !== 0) {
        let finalDeltaX = deltaX;
        let finalDeltaZ = deltaZ;
        
        if (!isMobile || !mobileJoystick.active) {
          const cos = Math.cos(cameraRotation.horizontal);
          const sin = Math.sin(cameraRotation.horizontal);
          
          finalDeltaX = deltaX * cos - deltaZ * sin;
          finalDeltaZ = deltaX * sin + deltaZ * cos;
        }
        
        setPlayerPosition(prev => {
          const newX = Math.max(-8, Math.min(8, prev.x + finalDeltaX));
          const newZ = Math.max(-8, Math.min(8, prev.z + finalDeltaZ));
          return { x: newX, z: newZ };
        });
      }
    };

    let lastTime = 0;
    const targetFPS = isMobile ? 30 : 60;
    const frameInterval = 1000 / targetFPS;
    
    const animate = (currentTime) => {
      frameId.current = requestAnimationFrame(animate);
      
      if (currentTime - lastTime < frameInterval) {
        return;
      }
      lastTime = currentTime;
      
      updatePlayerMovement();
      
      if (isFirstPerson) {
        camera.position.x = playerPosition.x;
        camera.position.y = 2;
        camera.position.z = playerPosition.z;
        
        const lookDirection = new THREE.Vector3(
          Math.sin(cameraRotation.horizontal) * Math.cos(cameraRotation.vertical),
          Math.sin(cameraRotation.vertical),
          Math.cos(cameraRotation.horizontal) * Math.cos(cameraRotation.vertical)
        );
        
        const lookTarget = new THREE.Vector3(
          playerPosition.x + lookDirection.x,
          2 + lookDirection.y,
          playerPosition.z + lookDirection.z
        );
        
        camera.lookAt(lookTarget);
      } else if (isMobile) {
        let rotationAngle;
        if (touchControls.autoRotate) {
          rotationAngle = Date.now() * 0.0005;
        } else {
          rotationAngle = touchControls.rotation;
        }
        
        camera.position.x = Math.cos(rotationAngle) * touchControls.zoom;
        camera.position.z = Math.sin(rotationAngle) * touchControls.zoom;
        camera.position.y = 8;
        camera.lookAt(0, 0, 0);
      } else {
        const time = Date.now() * 0.0005;
        camera.position.x = Math.cos(time) * 12;
        camera.position.z = Math.sin(time) * 12;
        camera.position.y = 8;
        camera.lookAt(0, 0, 0);
      }
      
      renderer.render(scene, camera);
    };

    animate(0);

    // РћР±СЂР°Р±РѕС‚РєР° РёР·РјРµРЅРµРЅРёСЏ СЂР°Р·РјРµСЂР° СЌРєСЂР°РЅР°
    const handleResize = () => {
      camera.aspect = currentMount.clientWidth / currentMount.clientHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(currentMount.clientWidth, currentMount.clientHeight);
    };
    
    window.addEventListener('resize', handleResize);

    // РћС‡РёСЃС‚РєР°
    return () => {
      window.removeEventListener('resize', handleResize);
      renderer.domElement.removeEventListener('mousemove', onMouseMove);
      renderer.domElement.removeEventListener('mousedown', onMouseDown);
      renderer.domElement.removeEventListener('mouseup', onMouseUp);
      renderer.domElement.removeEventListener('click', onMouseClick);
      document.removeEventListener('keydown', onKeyDown);
      document.removeEventListener('keyup', onKeyUp);
      
      // РњРѕР±РёР»СЊРЅС‹Рµ РѕР±СЂР°Р±РѕС‚С‡РёРєРё
      if (isMobile) {
        renderer.domElement.removeEventListener('touchstart', onTouchStart);
        renderer.domElement.removeEventListener('touchmove', onTouchMove);
        renderer.domElement.removeEventListener('touchend', onTouchEnd);
        renderer.domElement.removeEventListener('touchstart', onTouchStartForClick);
        renderer.domElement.removeEventListener('touchmove', onTouchMoveForClick);
        renderer.domElement.removeEventListener('touchend', onTouchEndForClick);
      }
      
      if (frameId.current) {
        cancelAnimationFrame(frameId.current);
      }
      if (currentMount && currentMount.contains(renderer.domElement)) {
        currentMount.removeChild(renderer.domElement);
      }
      renderer.dispose();
    };
  }, [isMobile, touchControls, isFirstPerson, playerPosition, cameraRotation, mobileJoystick]);

  // РћР±СЂР°Р±РѕС‚С‡РёРєРё РіРёСЂРѕСЃРєРѕРїР°
  useEffect(() => {
    if (!gyroEnabled || !isFirstPerson || !isMobile) return;

    const handleDeviceOrientation = (event) => {
      // РџРѕР»СѓС‡Р°РµРј СѓРіР»С‹ РїРѕРІРѕСЂРѕС‚Р° СѓСЃС‚СЂРѕР№СЃС‚РІР°
      const alpha = event.alpha || 0; // РџРѕРІРѕСЂРѕС‚ РІРѕРєСЂСѓРі РѕСЃРё Z (РєРѕРјРїР°СЃ)
      const beta = event.beta || 0;   // РџРѕРІРѕСЂРѕС‚ РІРѕРєСЂСѓРі РѕСЃРё X (РЅР°РєР»РѕРЅ РІРїРµСЂРµРґ/РЅР°Р·Р°Рґ)
      const gamma = event.gamma || 0; // РџРѕРІРѕСЂРѕС‚ РІРѕРєСЂСѓРі РѕСЃРё Y (РЅР°РєР»РѕРЅ РІР»РµРІРѕ/РІРїСЂР°РІРѕ)

      // РРЅРёС†РёР°Р»РёР·Р°С†РёСЏ РЅР°С‡Р°Р»СЊРЅС‹С… Р·РЅР°С‡РµРЅРёР№
      if (gyroRef.current.lastAlpha === 0) {
        gyroRef.current.lastAlpha = alpha;
        return;
      }

      // Р’С‹С‡РёСЃР»СЏРµРј РёР·РјРµРЅРµРЅРёРµ СѓРіР»Р° РїРѕРІРѕСЂРѕС‚Р°
      let deltaAlpha = alpha - gyroRef.current.lastAlpha;
      
      // РћР±СЂР°Р±РѕС‚РєР° РїРµСЂРµС…РѕРґР° С‡РµСЂРµР· 360 РіСЂР°РґСѓСЃРѕРІ
      if (deltaAlpha > 180) deltaAlpha -= 360;
      if (deltaAlpha < -180) deltaAlpha += 360;
      
      gyroRef.current.lastAlpha = alpha;

      // РћР±РЅРѕРІР»СЏРµРј РїРѕРІРѕСЂРѕС‚ РєР°РјРµСЂС‹ РЅР° РѕСЃРЅРѕРІРµ РіРёСЂРѕСЃРєРѕРїР°
      setCameraRotation(prev => ({
        horizontal: prev.horizontal - (deltaAlpha * Math.PI / 180) * 0.5,
        vertical: Math.max(-Math.PI/3, Math.min(Math.PI/3, -(beta - 90) * Math.PI / 180 * 0.3))
      }));
    };

    window.addEventListener('deviceorientation', handleDeviceOrientation);
    
    return () => {
      window.removeEventListener('deviceorientation', handleDeviceOrientation);
    };
  }, [gyroEnabled, isFirstPerson, isMobile]);

  const createPlant3D = (plotIndex, plot) => {
    if (!sceneRef.current || !plot) return;

    const scene = sceneRef.current;
    const seedData = seeds[plot.type];
    if (!seedData) return;
    
    if (plantMeshesRef.current[plotIndex]) {
      const oldPlant = plantMeshesRef.current[plotIndex];
      scene.remove(oldPlant);
      if (oldPlant.traverse) {
        oldPlant.traverse((child) => {
          if (child.geometry) child.geometry.dispose();
          if (child.material && child.material.dispose) child.material.dispose();
        });
      }
    }

    const group = new THREE.Group();
    const row = Math.floor(plotIndex / 3);
    const col = plotIndex % 3;
    group.position.set((col - 1) * 2, 0.2, (row - 1.5) * 2);
    
    group.userData = {
      stage: plot.stage,
      watered: plot.watered,
      type: plot.type
    };

    const plantColor = seedData.color;

    if (plot.stage === 0) {
      const seedGeometry = new THREE.BoxGeometry(0.1, 0.1, 0.1);
      const seedMaterial = new THREE.MeshBasicMaterial({ color: 0x8B4513 });
      const seedMesh = new THREE.Mesh(seedGeometry, seedMaterial);
      seedMesh.position.y = 0.05;
      group.add(seedMesh);
    } else if (plot.stage === 1) {
      const sproutGeometry = new THREE.ConeGeometry(0.1, 0.3, 3);
      const sproutMaterial = new THREE.MeshBasicMaterial({ color: 0x228B22 });
      const sproutMesh = new THREE.Mesh(sproutGeometry, sproutMaterial);
      sproutMesh.position.y = 0.15;
      group.add(sproutMesh);
    } else if (plot.stage === 2) {
      const stemGeometry = new THREE.CylinderGeometry(0.05, 0.05, 0.5, 6);
      const stemMaterial = new THREE.MeshBasicMaterial({ color: 0x228B22 });
      const stemMesh = new THREE.Mesh(stemGeometry, stemMaterial);
      stemMesh.position.y = 0.25;
      group.add(stemMesh);

      const leafGeometry = new THREE.ConeGeometry(0.2, 0.3, 4);
      const leafMaterial = new THREE.MeshBasicMaterial({ color: 0x32CD32 });
      const leafMesh = new THREE.Mesh(leafGeometry, leafMaterial);
      leafMesh.position.y = 0.5;
      group.add(leafMesh);
    } else if (plot.stage === 3) {
      const stemGeometry = new THREE.CylinderGeometry(0.08, 0.08, 0.8, 6);
      const stemMaterial = new THREE.MeshLambertMaterial({ color: 0x228B22 });
      const stemMesh = new THREE.Mesh(stemGeometry, stemMaterial);
      stemMesh.position.y = 0.4;
      if (!isMobile) stemMesh.castShadow = true;
      group.add(stemMesh);

      const fruitGeometry = new THREE.SphereGeometry(0.25, 6, 4);
      const fruitMaterial = new THREE.MeshLambertMaterial({ color: plantColor });
      const fruitMesh = new THREE.Mesh(fruitGeometry, fruitMaterial);
      fruitMesh.position.y = 0.8;
      if (!isMobile) fruitMesh.castShadow = true;
      group.add(fruitMesh);

      const leafCount = isMobile ? 2 : 4;
      for (let i = 0; i < leafCount; i++) {
        const leafGeometry = new THREE.ConeGeometry(0.15, 0.3, 3);
        const leafMaterial = new THREE.MeshBasicMaterial({ color: 0x32CD32 });
        const leafMesh = new THREE.Mesh(leafGeometry, leafMaterial);
        
        const angle = (i / leafCount) * Math.PI * 2;
        leafMesh.position.x = Math.cos(angle) * 0.2;
        leafMesh.position.z = Math.sin(angle) * 0.2;
        leafMesh.position.y = 0.6;
        leafMesh.rotation.z = angle;
        if (!isMobile) leafMesh.castShadow = true;
        
        group.add(leafMesh);
      }
    }

    if (!plot.watered && plot.stage > 0) {
      const waterGeometry = new THREE.SphereGeometry(0.1, 6, 4);
      const waterMaterial = new THREE.MeshBasicMaterial({ 
        color: 0x4169E1, 
        transparent: true, 
        opacity: 0.7 
      });
      const waterMesh = new THREE.Mesh(waterGeometry, waterMaterial);
      waterMesh.position.y = 1.2;
      group.add(waterMesh);
    }

    if (!isMobile) group.castShadow = true;
    
    scene.add(group);
    plantMeshesRef.current[plotIndex] = group;
    
    group.updateMatrixWorld(true);
  };

  useEffect(() => {
    garden.forEach((plot, index) => {
      const existingPlant = plantMeshesRef.current[index];
      
      if (plot) {
        const needsUpdate = !existingPlant || 
                           !existingPlant.userData || 
                           existingPlant.userData.stage !== plot.stage ||
                           existingPlant.userData.watered !== plot.watered;
        
        if (needsUpdate) {
          createPlant3D(index, plot);
        }
      } else if (existingPlant && sceneRef.current) {
        sceneRef.current.remove(existingPlant);
        existingPlant.traverse((child) => {
          if (child.geometry) child.geometry.dispose();
          if (child.material) child.material.dispose();
        });
        plantMeshesRef.current[index] = null;
      }
    });
  }, [garden]);

  useEffect(() => {
    const growthInterval = setInterval(() => {
      setGarden(prevGarden => {
        let hasChanges = false;
        const newGarden = prevGarden.map(plot => {
          if (plot && plot.watered && plot.stage < 3) {
            const timePassed = Date.now() - plot.plantedAt;
            const seedData = seeds[plot.type];
            if (!seedData) return plot;
            
            const requiredTime = seedData.growTime / 3;
            const nextStage = plot.stage + 1;
            
            if (timePassed > requiredTime * nextStage) {
              hasChanges = true;
              return { ...plot, stage: nextStage };
            }
          }
          return plot;
        });
        
        return hasChanges ? newGarden : prevGarden;
      });
    }, 5000);

    return () => clearInterval(growthInterval);
  }, []);

  // РђРІС‚РѕРјР°С‚РёС‡РµСЃРєРѕРµ РІРѕСЃСЃС‚Р°РЅРѕРІР»РµРЅРёРµ РІРѕРґС‹
  useEffect(() => {
    const waterInterval = setInterval(() => {
      setWaterAmount(prev => Math.min(prev + 1, 15));
    }, 10000);
    return () => clearInterval(waterInterval);
  }, []);

  const plantSeed = (plotIndex) => {
    const seedData = seeds[selectedSeed];
    if (coins >= seedData.cost && !garden[plotIndex]) {
      setCoins(prevCoins => prevCoins - seedData.cost);
      setGarden(prevGarden => {
        const newGarden = [...prevGarden];
        newGarden[plotIndex] = {
          id: `plant_${plotIndex}_${Date.now()}`,
          type: selectedSeed,
          stage: 0,
          plantedAt: Date.now(),
          watered: false
        };
        return newGarden;
      });
      addNotification(`рџЊ± РџРѕСЃР°РґРёР»Рё ${seedData.name}!`);
    }
  };

  const waterPlant = (plotIndex) => {
    if (waterAmount > 0 && garden[plotIndex] && !garden[plotIndex].watered) {
      setWaterAmount(prevWater => prevWater - 1);
      setGarden(prevGarden => {
        const newGarden = [...prevGarden];
        newGarden[plotIndex] = { 
          ...newGarden[plotIndex], 
          watered: true,
          wateredAt: Date.now()
        };
        return newGarden;
      });
      addNotification('рџ’§ Р Р°СЃС‚РµРЅРёРµ РїРѕР»РёС‚Рѕ!');
    }
  };

  const harvestPlant = (plotIndex) => {
    const plot = garden[plotIndex];
    if (plot && plot.stage === 3) {
      const seedData = seeds[plot.type];
      const reward = seedData.reward;
      
      setCoins(prevCoins => prevCoins + reward);
      setGarden(prevGarden => {
        const newGarden = [...prevGarden];
        newGarden[plotIndex] = null;
        return newGarden;
      });
      addNotification(`рџЋ‰ РЎРѕР±СЂР°Р»Рё ${seedData.name} Р·Р° ${reward} РјРѕРЅРµС‚!`);
    }
  };

  const handlePlotClick = (plotIndex) => {
    const plot = garden[plotIndex];
    if (!plot) {
      plantSeed(plotIndex);
    } else if (plot.stage === 3) {
      harvestPlant(plotIndex);
    } else if (!plot.watered) {
      waterPlant(plotIndex);
    }
  };

  const buyWater = () => {
    if (coins >= 50) {
      setCoins(prevCoins => prevCoins - 50);
      setWaterAmount(prevWater => Math.min(prevWater + 5, 15));
      addNotification('рџ’§ РљСѓРїРёР»Рё РІРѕРґСѓ!');
    }
  };

  // Р¤СѓРЅРєС†РёСЏ РІР·Р°РёРјРѕРґРµР№СЃС‚РІРёСЏ СЃ РіСЂСЏРґРєР°РјРё
  const handleInteraction = (clientX, clientY) => {
    if (!rendererRef.current || !cameraRef.current || !plotMeshesRef.current) return;
    
    const renderer = rendererRef.current;
    const camera = cameraRef.current;
    const plotMeshes = plotMeshesRef.current;

    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();

    const rect = renderer.domElement.getBoundingClientRect();
    mouse.x = ((clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((clientY - rect.top) / rect.height) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(plotMeshes);

    if (intersects.length > 0) {
      const plotIndex = intersects[0].object.userData.plotIndex;
      handlePlotClick(plotIndex);
    }
  };

  return (
    <div className="w-full h-full bg-gradient-to-b from-sky-300 to-green-300 relative" style={{ touchAction: 'none' }}>
      {/* 3D Canvas */}
      <div 
        ref={mountRef} 
        className="w-full h-2/3 bg-gradient-to-b from-sky-200 to-sky-400 rounded-lg shadow-2xl"
        style={{ minHeight: '400px' }}
      />

      {/* РњРѕР±РёР»СЊРЅС‹Рµ СЌР»РµРјРµРЅС‚С‹ СѓРїСЂР°РІР»РµРЅРёСЏ РІ СЂРµР¶РёРјРµ С…РѕРґСЊР±С‹ */}
      {isFirstPerson && isMobile && (
        <div 
          className="absolute inset-0 pointer-events-auto"
          onTouchStart={(e) => {
            // РўРѕР»СЊРєРѕ РµСЃР»Рё РєР°СЃР°РЅРёРµ РЅРµ РЅР° СЌР»РµРјРµРЅС‚Р°С… СѓРїСЂР°РІР»РµРЅРёСЏ
            if (e.target === e.currentTarget || e.target.closest('.control-area')) {
              e.preventDefault();
              const touch = e.touches[0];
              mouseRef.current.lastX = touch.clientX;
              mouseRef.current.lastY = touch.clientY;
              mouseRef.current.isDown = true;
            }
          }}
          onTouchMove={(e) => {
            if (mouseRef.current.isDown && (e.target === e.currentTarget || e.target.closest('.control-area'))) {
              e.preventDefault();
              const touch = e.touches[0];
              const deltaX = touch.clientX - mouseRef.current.lastX;
              const deltaY = touch.clientY - mouseRef.current.lastY;
              
              // РџРѕРІРѕСЂРѕС‚ РєР°РјРµСЂС‹ РїР°Р»СЊС†РµРј
              setCameraRotation(prev => ({
                horizontal: prev.horizontal - deltaX * 0.004,
                vertical: Math.max(-Math.PI/3, Math.min(Math.PI/3, prev.vertical - deltaY * 0.004))
              }));
              
              mouseRef.current.lastX = touch.clientX;
              mouseRef.current.lastY = touch.clientY;
            }
          }}
          onTouchEnd={(e) => {
            mouseRef.current.isDown = false;
          }}
          style={{ touchAction: 'none' }}
        >
          {/* Р’С‹Р±РѕСЂ СЃРµРјСЏРЅ СЃРІРµСЂС…Сѓ */}
          <div className="absolute top-4 left-4 right-4 pointer-events-auto control-area">
            <select
              value={selectedSeed}
              onChange={(e) => setSelectedSeed(e.target.value)}
              className="w-full p-3 rounded-lg bg-white/90 text-black font-semibold text-base border-2 border-green-400 shadow-lg"
              style={{ touchAction: 'manipulation' }}
            >
              {Object.entries(seeds).map(([key, seed]) => (
                <option key={key} value={key}>
                  {seed.name} - рџ’°{seed.cost} РјРѕРЅРµС‚
                </option>
              ))}
            </select>
          </div>

          {/* Р’РµСЂС‚РёРєР°Р»СЊРЅРѕРµ СѓРїСЂР°РІР»РµРЅРёРµ СЃРїСЂР°РІР° */}
          <div className="absolute right-4 top-1/2 transform -translate-y-1/2 pointer-events-auto control-area">
            <div className="flex flex-col items-center gap-6">
              
              {/* РљРЅРѕРїРєР° РІР·Р°РёРјРѕРґРµР№СЃС‚РІРёСЏ СЃРІРµСЂС…Сѓ */}
              <div className="flex flex-col items-center">
                <button
                  onTouchStart={(e) => {
                    e.preventDefault();
                    // РРјРёС‚РёСЂСѓРµРј РєР»РёРє РїРѕ С†РµРЅС‚СЂСѓ СЌРєСЂР°РЅР° РґР»СЏ РІР·Р°РёРјРѕРґРµР№СЃС‚РІРёСЏ
                    if (rendererRef.current) {
                      const rect = rendererRef.current.domElement.getBoundingClientRect();
                      const centerX = rect.left + rect.width / 2;
                      const centerY = rect.top + rect.height / 2;
                      handleInteraction(centerX, centerY);
                    }
                  }}
                  className="w-16 h-16 bg-green-600/80 rounded-full text-white text-2xl border-4 border-white/40 active:bg-green-700 flex items-center justify-center shadow-lg"
                  style={{ touchAction: 'none' }}
                >
                  рџЊ±
                </button>
                <div className="text-white/70 text-xs text-center mt-1">Р”РµР№СЃС‚РІРёРµ</div>
              </div>

              {/* РљРЅРѕРїРєР° РїРѕРєСѓРїРєРё РІРѕРґС‹ */}
              <div className="flex flex-col items-center">
                <button
                  onTouchStart={(e) => {
                    e.preventDefault();
                    buyWater();
                  }}
                  disabled={coins < 50}
                  className="w-14 h-14 bg-blue-600/80 rounded-full text-white text-xl border-4 border-white/40 active:bg-blue-700 disabled:bg-gray-500/80 flex items-center justify-center shadow-lg"
                  style={{ touchAction: 'none' }}
                >
                  рџ’§
                </button>
                <div className="text-white/70 text-xs text-center mt-1">Р’РѕРґР°</div>
              </div>

              {/* РљРЅРѕРїРєР° РіРёСЂРѕСЃРєРѕРїР° */}
              {gyroSupported && (
                <div className="flex flex-col items-center">
                  <button
                    onTouchStart={async (e) => {
                      e.preventDefault();
                      if (!gyroEnabled) {
                        const success = await requestGyroPermission();
                        if (success) {
                          addNotification('рџ“± Р“РёСЂРѕСЃРєРѕРї РІРєР»СЋС‡РµРЅ! РџРѕРІРѕСЂР°С‡РёРІР°Р№С‚Рµ С‚РµР»РµС„РѕРЅ');
                        } else {
                          addNotification('вќЊ РќРµ СѓРґР°Р»РѕСЃСЊ РІРєР»СЋС‡РёС‚СЊ РіРёСЂРѕСЃРєРѕРї');
                        }
                      } else {
                        setGyroEnabled(false);
                        addNotification('рџ“± Р“РёСЂРѕСЃРєРѕРї РІС‹РєР»СЋС‡РµРЅ');
                      }
                    }}
                    className={`w-14 h-14 rounded-full text-white text-xl border-4 border-white/40 flex items-center justify-center shadow-lg ${
                      gyroEnabled ? 'bg-blue-600/80 active:bg-blue-700' : 'bg-gray-600/80 active:bg-gray-700'
                    }`}
                    style={{ touchAction: 'none' }}
                  >
                    рџ“±
                  </button>
                  <div className="text-white/70 text-xs text-center mt-1">Р“РёСЂРѕ</div>
                </div>
              )}
            </div>
          </div>

          {/* Р’РёСЂС‚СѓР°Р»СЊРЅС‹Р№ РґР¶РѕР№СЃС‚РёРє РїРѕРґ РєРѕРЅС†РѕРј 3D canvas */}
          <div className="absolute left-6 pointer-events-auto control-area" style={{ top: 'calc(66.666667% + 10px)' }}>
            <div className="flex flex-col items-center">
              <div 
                ref={joystickRef}
                className="relative w-24 h-24 bg-black/40 rounded-full border-4 border-white/50 flex items-center justify-center shadow-lg"
                onTouchStart={(e) => {
                  e.preventDefault();
                  setMobileJoystick(prev => ({ ...prev, active: true }));
                }}
                onTouchMove={(e) => {
                  e.preventDefault();
                  if (joystickRef.current) {
                    const rect = joystickRef.current.getBoundingClientRect();
                    const centerX = rect.left + rect.width / 2;
                    const centerY = rect.top + rect.height / 2;
                    const touch = e.touches[0];
                    
                    const deltaX = touch.clientX - centerX;
                    const deltaY = touch.clientY - centerY;
                    const distance = Math.sqrt(deltaX * deltaX + deltaY * deltaY);
                    const maxDistance = rect.width / 2 - 10;
                    
                    if (distance <= maxDistance) {
                      setMobileJoystick({
                        x: deltaX / maxDistance,
                        y: deltaY / maxDistance,
                        active: true
                      });
                    } else {
                      const angle = Math.atan2(deltaY, deltaX);
                      setMobileJoystick({
                        x: Math.cos(angle),
                        y: Math.sin(angle),
                        active: true
                      });
                    }
                  }
                }}
                onTouchEnd={(e) => {
                  e.preventDefault();
                  setMobileJoystick({ x: 0, y: 0, active: false });
                }}
                style={{ touchAction: 'none' }}
              >
                {/* Р’РЅСѓС‚СЂРµРЅРЅРёР№ РєСЂСѓРі РґР¶РѕР№СЃС‚РёРєР° */}
                <div 
                  className="w-8 h-8 bg-white/90 rounded-full transition-transform duration-100 shadow-md"
                  style={{
                    transform: `translate(${mobileJoystick.x * 30}px, ${mobileJoystick.y * 30}px)`
                  }}
                />
              </div>
              <div className="text-white/80 text-sm text-center mt-2 font-semibold">Р”РІРёР¶РµРЅРёРµ</div>
            </div>
          </div>

          {/* РЎРєСЂС‹РІР°РµРј СЃС‚Р°СЂС‹Рµ РєРЅРѕРїРєРё */}
          <div style={{ display: 'none' }}>
          {/* РљРЅРѕРїРєР° РІР·Р°РёРјРѕРґРµР№СЃС‚РІРёСЏ */}
          <div className="absolute bottom-20 right-4 pointer-events-auto">
            <button
              onTouchStart={(e) => {
                e.preventDefault();
                // РРјРёС‚РёСЂСѓРµРј РєР»РёРє РїРѕ С†РµРЅС‚СЂСѓ СЌРєСЂР°РЅР° РґР»СЏ РІР·Р°РёРјРѕРґРµР№СЃС‚РІРёСЏ
                if (rendererRef.current) {
                  const rect = rendererRef.current.domElement.getBoundingClientRect();
                  const centerX = rect.left + rect.width / 2;
                  const centerY = rect.top + rect.height / 2;
                  handleInteraction(centerX, centerY);
                }
              }}
              className="w-16 h-16 bg-green-600/80 rounded-full text-white text-2xl border-4 border-white/40 active:bg-green-700 flex items-center justify-center"
              style={{ touchAction: 'none' }}
            >
              рџЊ±
            </button>
            <div className="text-white/70 text-xs text-center mt-2">Р”РµР№СЃС‚РІРёРµ</div>
          </div>

          </div>

          {/* РРЅРґРёРєР°С‚РѕСЂ РїСЂРёС†РµР»Р° */}
          <div className="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 pointer-events-none control-area">
            <div className="w-8 h-8 border-2 border-white/60 rounded-full flex items-center justify-center">
              <div className="w-2 h-2 bg-white/80 rounded-full"></div>
            </div>
          </div>

          {/* РњРёРЅРё-РєР°СЂС‚Р° - РїРµСЂРµРјРµС‰РµРЅР° РІС‹С€Рµ */}
          <div className="absolute top-16 right-4 w-16 h-16 bg-black/50 rounded-lg border-2 border-white/30 pointer-events-auto control-area">
            <div className="relative w-full h-full p-1">
              {/* РЎРµС‚РєР° РіСЂСЏРґРѕРє РЅР° РјРёРЅРё-РєР°СЂС‚Рµ */}
              {Array.from({ length: 12 }, (_, i) => {
                const row = Math.floor(i / 3);
                const col = i % 3;
                const plot = garden[i];
                return (
                  <div
                    key={i}
                    className={`absolute w-1 h-1 rounded-sm ${
                      plot ? (plot.stage === 3 ? 'bg-green-400' : 'bg-yellow-400') : 'bg-gray-600'
                    }`}
                    style={{
                      left: `${col * 25 + 20}%`,
                      top: `${row * 20 + 20}%`
                    }}
                  />
                );
              })}
              {/* РџРѕР·РёС†РёСЏ РёРіСЂРѕРєР° РЅР° РјРёРЅРё-РєР°СЂС‚Рµ */}
              <div
                className="absolute w-1.5 h-1.5 bg-blue-400 rounded-full"
                style={{
                  left: `${((playerPosition.x + 8) / 16) * 80 + 10}%`,
                  top: `${((playerPosition.z + 8) / 16) * 80 + 10}%`,
                  transform: 'translate(-50%, -50%)'
                }}
              />
            </div>
            <div className="text-white/70 text-xs text-center">РљР°СЂС‚Р°</div>
          </div>
        </div>
      )}

      {/* Р’РµСЂС…РЅСЏСЏ РїР°РЅРµР»СЊ - РІСЃРµ РєСЂРѕРјРµ СѓРїСЂР°РІР»РµРЅРёСЏ */}
      {!isFirstPerson && (
        <div className="absolute top-0 left-0 right-0 bg-gradient-to-b from-gray-900/90 to-transparent p-4 backdrop-blur-sm">
          {/* РЎС‚Р°С‚РёСЃС‚РёРєР° */}
          <div className="flex justify-center gap-6 mb-4">
            <div className="bg-yellow-500/90 px-4 py-2 rounded-full shadow-lg">
              <span className="text-white font-bold">рџ’° {coins} РјРѕРЅРµС‚</span>
            </div>
            <div className="bg-blue-500/90 px-4 py-2 rounded-full shadow-lg">
              <span className="text-white font-bold">рџ’§ {waterAmount} РІРѕРґС‹</span>
            </div>
          </div>

          {/* Р’С‹Р±РѕСЂ СЃРµРјСЏРЅ */}
          <div className="mb-4">
            <h3 className="text-white text-center mb-2 font-bold">Р’С‹Р±РµСЂРёС‚Рµ СЃРµРјРµРЅР°:</h3>
            <div className={`grid gap-2 ${isMobile ? 'grid-cols-2' : 'grid-cols-3'}`}>
              {Object.entries(seeds).map(([key, seed]) => (
                <button
                  key={key}
                  onClick={() => setSelectedSeed(key)}
                  className={`p-2 rounded-lg border-2 transition-all ${isMobile ? 'text-sm' : 'text-xs'} ${
                    selectedSeed === key
                      ? 'border-yellow-400 bg-yellow-500/80 text-white'
                      : 'border-gray-300 bg-white/80 hover:bg-white active:bg-gray-200'
                  }`}
                  style={{ touchAction: 'manipulation' }}
                >
                  <div className="font-semibold">{seed.name}</div>
                  <div className="text-gray-600">рџ’° {seed.cost}</div>
                </button>
              ))}
            </div>
          </div>

          {/* РРЅСЃС‚СЂСѓРєС†РёРё */}
          <div className="text-center text-white/80 text-sm">
            {isMobile ? (
              <div>
                <p>рџ‘† РљРѕСЃРЅРёС‚РµСЃСЊ РіСЂСЏРґРєРё: РїРѕСЃР°РґРёС‚СЊ в†’ РїРѕР»РёС‚СЊ в†’ СЃРѕР±СЂР°С‚СЊ</p>
                <p>рџ”„ РћРґРёРЅ РїР°Р»РµС† - РїРѕРІРѕСЂРѕС‚, РґРІР° РїР°Р»СЊС†Р° - Р·СѓРј</p>
                <p>рџљ¶вЂЌв™‚пёЏ РљРЅРѕРїРєР° "Р’РѕР№С‚Рё РІ СЃР°Рґ" - СЂРµР¶РёРј С…РѕРґСЊР±С‹</p>
              </div>
            ) : (
              <div>
                <p>рџ–±пёЏ РљР»РёРєРЅРёС‚Рµ РїРѕ РіСЂСЏРґРєРµ: РїРѕСЃР°РґРёС‚СЊ в†’ РїРѕР»РёС‚СЊ в†’ СЃРѕР±СЂР°С‚СЊ</p>
                <p>рџ“· РљР°РјРµСЂР° Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРё РІСЂР°С‰Р°РµС‚СЃСЏ РІРѕРєСЂСѓРі СЃР°РґР°</p>
                <p>вЊЁпёЏ F РёР»Рё РєРЅРѕРїРєР° - РІРѕР№С‚Рё РІ СЃР°Рґ</p>
              </div>
            )}
          </div>
        </div>
      )}

      {/* РљРѕРјРїР°РєС‚РЅР°СЏ СЃС‚Р°С‚РёСЃС‚РёРєР° РґР»СЏ СЂРµР¶РёРјР° РїРµСЂРІРѕРіРѕ Р»РёС†Р° */}
      {isFirstPerson && (
        <div className="absolute top-16 left-4 right-4 flex justify-center gap-4 pointer-events-none">
          <div className="bg-yellow-500/90 px-3 py-1 rounded-full shadow-lg text-sm">
            <span className="text-white font-bold">рџ’° {coins}</span>
          </div>
          <div className="bg-blue-500/90 px-3 py-1 rounded-full shadow-lg text-sm">
            <span className="text-white font-bold">рџ’§ {waterAmount}</span>
          </div>
        </div>
      )}

      {/* РќРёР¶РЅСЏСЏ РїР°РЅРµР»СЊ - С‚РѕР»СЊРєРѕ СѓРїСЂР°РІР»РµРЅРёРµ */}
      <div className="absolute bottom-0 left-0 right-0 bg-gradient-to-t from-gray-900/90 to-transparent p-4">


        {/* РљРЅРѕРїРєРё СѓРїСЂР°РІР»РµРЅРёСЏ */}
        <div className={`flex justify-center gap-2 ${isMobile && isFirstPerson ? 'flex-wrap' : 'gap-4'}`}>
          {!(isFirstPerson && isMobile) && (
            <button
              onClick={buyWater}
              disabled={coins < 50}
              className="px-4 py-2 bg-blue-600 text-white rounded-lg disabled:bg-gray-500 hover:bg-blue-700 active:bg-blue-800 transition-all shadow-lg"
              style={{ touchAction: 'manipulation' }}
            >
              рџ’§ РљСѓРїРёС‚СЊ РІРѕРґСѓ (50 РјРѕРЅРµС‚)
            </button>
          )}
          <button
            onClick={() => setIsFirstPerson(!isFirstPerson)}
            className={`px-4 py-2 text-white rounded-lg transition-all shadow-lg ${
              isFirstPerson ? 'bg-purple-600 hover:bg-purple-700' : 'bg-green-600 hover:bg-green-700'
            }`}
            style={{ touchAction: 'manipulation' }}
          >
            {isFirstPerson ? 'рџЋ® Р’С‹Р№С‚Рё' : 'рџљ¶вЂЌв™‚пёЏ Р’РѕР№С‚Рё РІ СЃР°Рґ'}
          </button>
          {isFirstPerson && isMobile && (
            <button
              onClick={buyWater}
              disabled={coins < 50}
              className="px-3 py-2 bg-blue-600 text-white rounded-lg disabled:bg-gray-500 active:bg-blue-800 transition-all shadow-lg text-sm"
              style={{ touchAction: 'manipulation' }}
            >
              рџ’§ Р’РѕРґР°
            </button>
          )}
          {isMobile && !isFirstPerson && (
            <button
              onClick={() => setTouchControls(prev => ({ ...prev, autoRotate: !prev.autoRotate }))}
              className={`px-4 py-2 text-white rounded-lg transition-all shadow-lg ${
                touchControls.autoRotate ? 'bg-green-600 hover:bg-green-700' : 'bg-gray-600 hover:bg-gray-700'
              }`}
              style={{ touchAction: 'manipulation' }}
            >
              {touchControls.autoRotate ? 'вЏёпёЏ РЎС‚РѕРї' : 'в–¶пёЏ РђРІС‚Рѕ'}
            </button>
          )}
        </div>

        {/* РРЅСЃС‚СЂСѓРєС†РёРё РґР»СЏ СЂРµР¶РёРјР° РїРµСЂРІРѕРіРѕ Р»РёС†Р° */}
        {isFirstPerson && (
          <div className="text-center text-white/80 text-sm mt-2">
            {isMobile ? (
              <div>
                <p>рџ•№пёЏ Р”Р¶РѕР№СЃС‚РёРє - РґРІРёР¶РµРЅРёРµ, рџЊ± РєРЅРѕРїРєР° - РґРµР№СЃС‚РІРёРµ</p>
                <p>рџ‘† Р’РѕРґРёС‚Рµ РїР°Р»СЊС†РµРј РїРѕ СЌРєСЂР°РЅСѓ - РїРѕРІРѕСЂРѕС‚ РіРѕР»РѕРІС‹ {gyroSupported && 'рџ“± РіРёСЂРѕСЃРєРѕРї'}</p>
              </div>
            ) : (
              <div>
                <p>рџљ¶вЂЌв™‚пёЏ WASD РёР»Рё СЃС‚СЂРµР»РєРё - С…РѕРґСЊР±Р°, РјС‹С€СЊ - РІР·РіР»СЏРґ</p>
                <p>рџ–±пёЏ Р—Р°Р¶РјРёС‚Рµ Р»РµРІСѓСЋ РєРЅРѕРїРєСѓ РјС‹С€Рё РґР»СЏ РїРѕРІРѕСЂРѕС‚Р°</p>
              </div>
            )}
          </div>
        )}
      </div>

      {/* РЈРІРµРґРѕРјР»РµРЅРёСЏ */}
      {notifications.length > 0 && (
        <div className="fixed top-4 right-4 z-50 space-y-2">
          {notifications.map(notification => (
            <div
              key={notification.id}
              className="bg-green-600 text-white px-4 py-2 rounded-lg shadow-xl animate-bounce backdrop-blur-sm border border-green-400"
            >
              {notification.message}
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default Garden3D;
