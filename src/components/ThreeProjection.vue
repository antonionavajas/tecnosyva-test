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
            const geometry = new THREE.PlaneGeometry(10, 10);
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

            // Convertir a puntos 3D con Z inicial alta
            const points3D = fireSurface.map(
                (point) => new THREE.Vector3(point.x, point.y)
            );

            // Raycaster para calcular intersecciones
            const raycaster = new THREE.Raycaster();
            const intersections: THREE.Vector3[] = [];

            points3D.forEach((point) => {
                raycaster.set(point, new THREE.Vector3(0, 0, -1));
                const hits = raycaster.intersectObject(planeMesh);
                if (hits.length > 0) {
                    intersections.push(hits[0].point);
                }
            });

            // Si hay suficientes puntos de intersección, crear una nueva superficie
            if (intersections.length > 2) {
                const shape = new THREE.Shape(
                    intersections.map((p) => new THREE.Vector2(p.x, p.y))
                );
                const shapeGeometry = new THREE.ShapeGeometry(shape);
                const shapeMaterial = new THREE.MeshBasicMaterial({
                    color: 0xff0000,
                    side: THREE.DoubleSide,
                    wireframe: true,
                });
                const surfaceMesh = new THREE.Mesh(
                    shapeGeometry,
                    shapeMaterial
                );
                scene.add(surfaceMesh);
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
