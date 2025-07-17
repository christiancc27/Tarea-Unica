import streamlit as st
import pandas as pd
import plotly.express as px

# --- Configuración de la aplicación ---
st.set_page_config(
    page_title="Auditoría COBIT 2019",
    page_icon="📊",
    layout="wide"
)

st.title("📊 Auditoría de Sistemas basada en COBIT 2019")
st.write("Sube tu archivo Excel con las preguntas de auditoría para comenzar.")

# --- Definición del sistema de puntuación y mensajes ---
interpretacion_riesgo = {
    "Riesgo Alto": {
        "rango": (1.0, 2.0),
        "color": "red",
        "emoji": "🔴",
        "mensaje": "¡Atención! Riesgo Alto identificado. El promedio de cumplimiento en este dominio indica deficiencias significativas. Es crucial implementar acciones correctivas de manera urgente para mitigar riesgos potenciales que podrían impactar seriamente los objetivos de negocio y la seguridad de la información. Se recomienda una revisión profunda de los procesos y controles."
    },
    "Riesgo Medio": {
        "rango": (2.1, 3.5),
        "color": "orange",
        "emoji": "🟠",
        "mensaje": "Oportunidad de Mejora detectada. Este dominio muestra un cumplimiento parcial. Si bien existen controles, hay áreas que necesitan fortalecimiento para optimizar el rendimiento y reducir vulnerabilidades. Considere implementar nuevas prácticas o refinar las existentes para alcanzar un nivel de madurez más robusto y asegurar una mayor alineación con los estándares."
    },
    "Cumplimiento Bueno": {
        "rango": (3.6, 5.0),
        "color": "green",
        "emoji": "🟢",
        "mensaje": "¡Excelente! Cumplimiento Bueno. El promedio en este dominio refleja que los controles y procesos están bien establecidos y son efectivos. La gestión de riesgos es adecuada y se están alcanzando los objetivos deseados. Se recomienda mantener este nivel de control y explorar oportunidades para la mejora continua, buscando la excelencia y la superación de expectativas para asegurar la resiliencia y el valor a largo plazo."
    }
}

# --- Subir archivo Excel ---
uploaded_file = st.file_uploader("Sube tu archivo de preguntas (Excel)", type=["xlsx"])

if uploaded_file:
    try:
        df = pd.read_excel(uploaded_file)
        if not all(col in df.columns for col in ["Dominio", "Pregunta"]):
            st.error("El archivo Excel debe contener las columnas 'Dominio' y 'Pregunta'.")
        else:
            st.success("Archivo cargado exitosamente. ¡Comienza la auditoría!")
            st.write("Por favor, responde a cada pregunta usando el deslizador de 1 a 5.")

            respuestas = []
            st.session_state.respuestas_dict = {} # Para almacenar respuestas por ID de pregunta

            # Agrupar preguntas por dominio para una mejor visualización
            dominios = df["Dominio"].unique()
            
            for dominio in dominios:
                st.subheader(f"Dominio: {dominio}")
                preguntas_dominio = df[df["Dominio"] == dominio]
                
                for idx, row in preguntas_dominio.iterrows():
                    pregunta_id = f"{row['Dominio']}_{idx}" # Generar un ID único para la pregunta
                    
                    # Usar un valor por defecto si no hay respuesta previa, o la respuesta guardada
                    current_value = st.session_state.respuestas_dict.get(pregunta_id, 3) 
                    
                    valor = st.slider(
                        f"**{row['Pregunta']}**",
                        min_value=1,
                        max_value=5,
                        value=current_value,
                        step=1,
                        key=pregunta_id # Asignar una key única para cada slider
                    )
                    st.session_state.respuestas_dict[pregunta_id] = valor # Guardar la respuesta en el estado de la sesión
                    respuestas.append({
                        "Dominio": row["Dominio"],
                        "Pregunta": row["Pregunta"],
                        "Respuesta": valor
                    })

            st.markdown("---")

            if st.button("Generar Informe de Auditoría", type="primary"):
                if not respuestas:
                    st.warning("Por favor, responde al menos una pregunta antes de generar el informe.")
                else:
                    df_resp = pd.DataFrame(respuestas)
                    resumen = df_resp.groupby("Dominio")["Respuesta"].mean().reset_index()
                    resumen.columns = ["Dominio", "Promedio"]

                    st.header("Resultados de la Auditoría por Dominio")
                    st.dataframe(resumen.set_index("Dominio"))

                    st.markdown("---")

                    col1, col2 = st.columns([1, 1])

                    with col1:
                        st.subheader("Visualización General del Cumplimiento")
                        # Radar chart o gráfico de barras
                        fig_radar = px.line_polar(
                            resumen,
                            r='Promedio',
                            theta='Dominio',
                            line_close=True,
                            range_r=[0, 5],
                            title="Nivel de Cumplimiento por Dominio (Radar Chart)"
                        )
                        fig_radar.update_traces(fill='toself')
                        st.plotly_chart(fig_radar)
                    
                    with col2:
                        st.subheader("Indicadores de Cumplimiento (Semáforo)")
                        for _, row in resumen.iterrows():
                            promedio = row["Promedio"]
                            dominio_nombre = row["Dominio"]

                            interpretacion_encontrada = None
                            for key, info in interpretacion_riesgo.items():
                                if info["rango"][0] <= promedio <= info["rango"][1]:
                                    interpretacion_encontrada = info
                                    break
                            
                            if interpretacion_encontrada:
                                st.markdown(f"### {interpretacion_encontrada['emoji']} **{dominio_nombre}: {interpretacion_encontrada['rango'][0]:.1f} - {interpretacion_encontrada['rango'][1]:.1f} | Promedio: {promedio:.2f}**")
                                st.markdown(f"**Interpretación:** *{interpretacion_encontrada['mensaje']}*")
                                st.markdown("---")
                            else:
                                st.info(f"{dominio_nombre}: Promedio {promedio:.2f}. No se encontró una interpretación específica para este rango.")

    except Exception as e:
        st.error(f"Ocurrió un error al procesar el archivo: {e}")

st.sidebar.markdown("---")
st.sidebar.info("Esta aplicación te ayuda a realizar una auditoría de sistemas basada en COBIT 2019 de manera interactiva.")