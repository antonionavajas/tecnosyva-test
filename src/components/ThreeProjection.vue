<script lang="ts">
import { defineComponent, onMounted, ref } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";

export default defineComponent({
    name: "ThreeProjection",
    setup() {
        const container = ref<HTMLDivElement | null>(null);
        const axesHelper = new THREE.AxesHelper(5); // Tamaño del gizmo (longitud de los ejes)
        onMounted(() => {
            if (!container.value) return;

            // Escena, cámara y renderizador
            const scene = new THREE.Scene();
            const camera = new THREE.PerspectiveCamera(
                75,
                container.value.clientWidth / container.value.clientHeight,
                0.1,
                1000
            );
            scene.add(axesHelper);

            const renderer = new THREE.WebGLRenderer();
            renderer.setSize(
                container.value.clientWidth,
                container.value.clientHeight
            );
            container.value.appendChild(renderer.domElement);

            // Añadir una malla como plano
            const geometry = new THREE.BoxGeometry(10, 10, 0.1);
            const material = new THREE.MeshBasicMaterial({
                color: 0x00ff00,
                side: THREE.DoubleSide,
                wireframe: true,
            });
            const planeMesh = new THREE.Mesh(geometry, material);
            planeMesh.rotation.x = Math.PI / 2; // Girar para que sea horizontal
            scene.add(planeMesh);

            // Configurar la cámara
            camera.position.set(0, 5, 10);
            camera.lookAt(0, 0, 0);

            // OrbitControls para manipular la cámara
            const controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true; // Suaviza los movimientos
            controls.dampingFactor = 0.05;
            controls.enablePan = false; // Desactivar el movimiento de pan
            controls.enableZoom = true; // Permitir zoom

            controls.maxPolarAngle = Math.PI / 2; // Fijar la cámara horizontalmente
            // A recordar, X Y Z, rojo, amarillo, azul en el gizmo
            // Puntos 2D del polígono
            const fireSurface = [
                new THREE.Vector3(-2, 3, -2),
                new THREE.Vector3(2, 3, -2),
                new THREE.Vector3(3, 3, 0),
                new THREE.Vector3(2, 3, 2),
                new THREE.Vector3(-2, 3, 2),
                new THREE.Vector3(-2, 3, -2),
            ];

            // Parece que hay que convertir los puntos a un array Float32 para que funcione el BufferGeometry
            const fireSurfaceFlat = new Float32Array(
                fireSurface.flatMap((p) => [p.x, p.y, p.z])
            );

            const geometryFire = new THREE.BufferGeometry();
            geometryFire.setAttribute(
                "position",
                new THREE.BufferAttribute(fireSurfaceFlat, 3)
            );

            const fireSurfaceMaterial = new THREE.LineBasicMaterial({
                color: 0xff0000,
            });

            const fireSurfaceMesh = new THREE.Line(
                geometryFire,
                fireSurfaceMaterial
            );

            scene.add(fireSurfaceMesh);

            // Convertir a puntos 3D con Y inicial alta
            const points3D = fireSurface.map(
                (point) => new THREE.Vector3(point.x, 10, point.z)
            );

            // Raycaster para calcular intersecciones

            const intersections: THREE.Vector3[] = [];

            points3D.forEach((point) => {
                let raycaster = new THREE.Raycaster();
                let raycastDestinationPoint = new THREE.Vector3(0, -1, 0);

                raycaster.set(point, raycastDestinationPoint);

                const length = 20; // Longitud de la línea

                const hits = raycaster.intersectObject(planeMesh);
                console.log("hit!", hits);
                let material = new THREE.LineBasicMaterial({
                    color: 0xff0000,
                });
                if (hits.length > 0) {
                    intersections.push(hits[1].point);
                    material = new THREE.LineBasicMaterial({
                        color: 0x00ff00,
                    });
                }

                const origin = point; // Origen de la línea
                const direction = raycastDestinationPoint.normalize(); // Direcci
                const endPoint = origin
                    .clone()
                    .add(direction.multiplyScalar(length));
                const geometry = new THREE.BufferGeometry().setFromPoints([
                    origin,
                    endPoint,
                ]);

                const line = new THREE.Line(geometry, material);
                scene.add(line);
            });
            console.log(intersections);
            // Si hay suficientes puntos de intersección, crear una nueva superficie
            if (intersections.length > 2) {
                intersections.map((point) => {
                    const sphere = new THREE.Mesh(
                        new THREE.SphereGeometry(15, 32, 16),
                        new THREE.MeshBasicMaterial({ color: 0x0000ff })
                    );
                    sphere.position.copy(point);

                    console.log(sphere.position);
                    scene.add(sphere);
                });
            }

            // Animación
            const animate = () => {
                requestAnimationFrame(animate);
                controls.update();
                renderer.render(scene, camera);
            };
            animate();

            // TODO: Hook
            window.addEventListener("resize", () => {
                if (!container.value) return;
                camera.aspect =
                    container.value.clientWidth / container.value.clientHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(
                    container.value.clientWidth,
                    container.value.clientHeight
                );
            });
        });

        return {
            container,
        };
    },
});
</script>

<template>
    <div ref="container" style="width: 100vw; height: 100vh"></div>
</template>

<style scoped>
div {
    width: 100%;
    height: 100vh;
    overflow: hidden;
}
</style>
