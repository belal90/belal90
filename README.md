مستخرج صفحات PDF

<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>مستخرج صفحات PDF</title>
    <meta name="description" content="تطبيق ويب لرفع ملف PDF، تحديد صفحات معينة (متفرقة أو متتابعة)، واستخراجها في ملف PDF جديد جاهز للتنزيل." />
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/pdf-lib/dist/pdf-lib.min.js"></script>
    <script src="https://unpkg.com/react@18/umd/react.production.min.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js" crossorigin></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
</head>
<body class="bg-slate-100">
    <div id="root"></div>
    <script type="text/babel">
        const { useState, useEffect, useRef, useCallback } = React;

        // From components/Icons.tsx
        const UploadIcon = ({ className }) => (
            <svg xmlns="http://www.w3.org/2000/svg" className={className} fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" />
            </svg>
        );

        const PdfFileIcon = ({ className }) => (
            <svg xmlns="http://www.w3.org/2000/svg" className={className} fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
            </svg>
        );

        const DownloadIcon = ({ className }) => (
          <svg xmlns="http://www.w3.org/2000/svg" className={className} fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4" />
          </svg>
        );

        const XCircleIcon = ({ className }) => (
          <svg xmlns="http://www.w3.org/2000/svg" className={className} fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M10 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z" />
          </svg>
        );

        const SpinnerIcon = ({ className }) => (
          <svg className={`${className} animate-spin`} xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
            <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
        );

        // From services/pdfUtils.ts
        const parsePageRanges = (pageString, totalPages) => {
            const pageSet = new Set();
            if (!pageString.trim()) {
                throw new Error("الرجاء إدخال أرقام الصفحات.");
            }

            const parts = pageString.split(',').map(part => part.trim());

            for (const part of parts) {
                if (part.includes('-')) {
                    const [start, end] = part.split('-').map(num => parseInt(num.trim(), 10));
                    if (isNaN(start) || isNaN(end) || start <= 0 || end <= 0 || start > end || end > totalPages) {
                        throw new Error(`نطاق الصفحات غير صالح: "${part}". تأكد أن الأرقام ضمن حدود الملف (1-${totalPages}).`);
                    }
                    for (let i = start; i <= end; i++) {
                        pageSet.add(i - 1); // 0-based index
                    }
                } else {
                    const page = parseInt(part, 10);
                    if (isNaN(page) || page <= 0 || page > totalPages) {
                         throw new Error(`رقم الصفحة غير صالح: "${part}". تأكد أن الرقم ضمن حدود الملف (1-${totalPages}).`);
                    }
                    pageSet.add(page - 1); // 0-based index
                }
            }

            if (pageSet.size === 0) {
                throw new Error("لم يتم تحديد صفحات صالحة.");
            }

            return Array.from(pageSet).sort((a, b) => a - b);
        };

        // From components/FileUpload.tsx
        const FileUpload = ({ onFileSelect, onError }) => {
            const [isDragging, setIsDragging] = useState(false);

            const handleFile = useCallback((file) => {
                if (file) {
                    if (file.type === "application/pdf") {
                        onFileSelect(file);
                        onError(""); // Clear previous errors
                    } else {
                        onError("خطأ: الملف المرفوع ليس من نوع PDF.");
                    }
                }
            }, [onFileSelect, onError]);

            const handleDragEnter = (e) => {
                e.preventDefault();
                e.stopPropagation();
                setIsDragging(true);
            };

            const handleDragLeave = (e) => {
                e.preventDefault();
                e.stopPropagation();
                setIsDragging(false);
            };

            const handleDragOver = (e) => {
                e.preventDefault();
                e.stopPropagation();
            };

            const handleDrop = (e) => {
                e.preventDefault();
                e.stopPropagation();
                setIsDragging(false);
                const file = e.dataTransfer.files && e.dataTransfer.files[0];
                handleFile(file);
            };

            const handleFileChange = (e) => {
                const file = e.target.files && e.target.files[0];
                handleFile(file);
            };

            return (
                <div 
                    className={`w-full p-8 border-2 border-dashed rounded-lg text-center transition-colors duration-200 ease-in-out ${
                        isDragging ? 'border-sky-500 bg-sky-50' : 'border-slate-300 bg-white'
                    }`}
                    onDragEnter={handleDragEnter}
                    onDragLeave={handleDragLeave}
                    onDragOver={handleDragOver}
                    onDrop={handleDrop}
                >
                    <input 
                        type="file" 
                        id="file-upload" 
                        className="hidden" 
                        accept="application/pdf"
                        onChange={handleFileChange}
                    />
                    <label htmlFor="file-upload" className="cursor-pointer flex flex-col items-center justify-center">
                        <UploadIcon className="w-12 h-12 text-slate-400 mb-4" />
                        <p className="text-slate-700 font-semibold">
                            اسحب وأفلت ملف PDF هنا، أو 
                            <span className="text-sky-600"> تصفح الملفات</span>
                        </p>
                        <p className="text-xs text-slate-500 mt-1">يتم التعامل مع الملفات محلياً على جهازك</p>
                    </label>
                </div>
            );
        };

        // From App.tsx
        const App = () => {
            const [pdfFile, setPdfFile] = useState(null);
            const [totalPages, setTotalPages] = useState(0);
            const [pageInput, setPageInput] = useState('');
            const [outputFileName, setOutputFileName] = useState('');
            const [isProcessing, setIsProcessing] = useState(false);
            const [error, setError] = useState('');
            const [processedPdfUrl, setProcessedPdfUrl] = useState(null);
            const workerRef = useRef(null);
            
            const workerScript = `
                importScripts('https://cdn.jsdelivr.net/npm/pdf-lib/dist/pdf-lib.min.js');
                const { PDFDocument } = self.PDFLib;

                const getPdfPageCount = async (pdfFile) => {
                    const existingPdfBytes = await pdfFile.arrayBuffer();
                    const pdfDoc = await PDFDocument.load(existingPdfBytes);
                    return pdfDoc.getPageCount();
                };

                const extractPdfPages = async (pdfFile, pageIndices) => {
                    const existingPdfBytes = await pdfFile.arrayBuffer();
                    const pdfDoc = await PDFDocument.load(existingPdfBytes);
                    
                    const newPdfDoc = await PDFDocument.create();
                    
                    const copiedPages = await newPdfDoc.copyPages(pdfDoc, pageIndices);
                    copiedPages.forEach((page) => {
                        newPdfDoc.addPage(page);
                    });

                    const newPdfBytes = await newPdfDoc.save();
                    return newPdfBytes;
                };

                self.onmessage = async (event) => {
                    const { type, payload } = event.data;

                    try {
                        if (type === 'GET_PAGE_COUNT') {
                            const { file } = payload;
                            const count = await getPdfPageCount(file);
                            self.postMessage({ type: 'PAGE_COUNT_RESULT', payload: { count } });
                        } else if (type === 'EXTRACT_PAGES') {
                            const { file, pageIndices } = payload;
                            const pdfBytes = await extractPdfPages(file, pageIndices);
                            self.postMessage({ type: 'EXTRACT_RESULT', payload: { pdfBytes } }, [pdfBytes.buffer]);
                        }
                    } catch (err) {
                        self.postMessage({ 
                            type: 'ERROR', 
                            payload: { 
                                message: err.message || 'An unknown error occurred in the worker.',
                                originalType: type
                            } 
                        });
                    }
                };
            `;

            useEffect(() => {
                const blob = new Blob([workerScript], { type: 'application/javascript' });
                const workerUrl = URL.createObjectURL(blob);
                const worker = new Worker(workerUrl);
                workerRef.current = worker;

                worker.onmessage = (event) => {
                    const { type, payload } = event.data;
                    switch (type) {
                        case 'PAGE_COUNT_RESULT':
                            setTotalPages(payload.count);
                            break;
                        case 'EXTRACT_RESULT':
                            const blob = new Blob([payload.pdfBytes], { type: 'application/pdf' });
                            const url = URL.createObjectURL(blob);
                            setProcessedPdfUrl(url);
                            setIsProcessing(false);
                            break;
                        case 'ERROR':
                            const message = payload.message || 'حدث خطأ غير متوقع أثناء المعالجة.';
                            if (payload.originalType === 'GET_PAGE_COUNT') {
                                setError(`لا يمكن قراءة ملف PDF: ${message}`);
                                setPdfFile(null);
                            } else {
                                setError(message);
                                setIsProcessing(false);
                            }
                            break;
                    }
                };
                
                worker.onerror = (e) => {
                    setError(`حدث خطأ في العامل: ${e.message}`);
                    setIsProcessing(false);
                };

                return () => {
                    worker.terminate();
                    URL.revokeObjectURL(workerUrl);
                };
            }, []);

            useEffect(() => {
                if (pdfFile) {
                    setOutputFileName(`${pdfFile.name.replace(/\.pdf$/i, '')}-مستخرج.pdf`);
                    setTotalPages(0);
                     if (workerRef.current) {
                        workerRef.current.postMessage({
                            type: 'GET_PAGE_COUNT',
                            payload: { file: pdfFile }
                        });
                    }
                }
            }, [pdfFile]);
            
            useEffect(() => {
                return () => {
                    if (processedPdfUrl) {
                        URL.revokeObjectURL(processedPdfUrl);
                    }
                };
            }, [processedPdfUrl]);

            const handleFileSelect = (file) => {
                resetState();
                setPdfFile(file);
            };

            const resetState = () => {
                setPdfFile(null);
                setTotalPages(0);
                setPageInput('');
                setOutputFileName('');
                setIsProcessing(false);
                setError('');
                if (processedPdfUrl) {
                    URL.revokeObjectURL(processedPdfUrl);
                }
                setProcessedPdfUrl(null);
            };

            const handleExtract = async () => {
                if (!pdfFile || !pageInput || !outputFileName) {
                    setError('الرجاء رفع ملف وتحديد الصفحات.');
                    return;
                }

                setError('');

                if (totalPages === 0) {
                    setError("لم يتم حساب عدد الصفحات بعد، الرجاء الانتظار.");
                    return;
                }

                let pageIndices;
                try {
                    pageIndices = parsePageRanges(pageInput, totalPages);
                } catch (err) {
                    setError(err.message);
                    return;
                }

                setIsProcessing(true);
                if (processedPdfUrl) {
                    URL.revokeObjectURL(processedPdfUrl);
                    setProcessedPdfUrl(null);
                }

                if (workerRef.current) {
                    workerRef.current.postMessage({
                        type: 'EXTRACT_PAGES',
                        payload: { file: pdfFile, pageIndices }
                    });
                }
            };
            
            return (
                <div className="min-h-screen flex items-center justify-center p-4 bg-slate-100 font-sans">
                    <div className="w-full max-w-2xl mx-auto">
                        <header className="text-center mb-8">
                            <h1 className="text-4xl font-bold text-slate-800">مستخرج صفحات PDF</h1>
                            <p className="text-slate-600 mt-2">استخرج صفحات محددة من ملف PDF بسهولة وأمان.</p>
                        </header>

                        <main className="bg-white shadow-xl rounded-lg p-8 space-y-6">
                            {!pdfFile ? (
                                <FileUpload onFileSelect={handleFileSelect} onError={setError} />
                            ) : (
                                <div className="space-y-6">
                                    <div className="flex items-center justify-between p-4 border border-slate-200 rounded-lg bg-slate-50">
                                        <div className="flex items-center gap-3 overflow-hidden">
                                            <PdfFileIcon className="w-10 h-10 text-red-500 flex-shrink-0" />
                                            <div className="flex-grow overflow-hidden">
                                              <p className="font-semibold text-slate-800 truncate" title={pdfFile.name}>{pdfFile.name}</p>
                                              <p className="text-sm text-slate-500">{totalPages > 0 ? `${totalPages} صفحة` : 'جاري حساب الصفحات...'}</p>
                                            </div>
                                        </div>
                                        <button onClick={resetState} className="p-1 text-slate-400 hover:text-slate-600 rounded-full focus:outline-none focus:ring-2 focus:ring-slate-400">
                                            <XCircleIcon className="w-6 h-6" />
                                        </button>
                                    </div>
                                    
                                    <div>
                                        <label htmlFor="pages" className="block text-sm font-medium text-slate-700 mb-1">
                                            أرقام الصفحات المراد استخراجها
                                        </label>
                                        <input
                                            type="text"
                                            id="pages"
                                            value={pageInput}
                                            onChange={(e) => setPageInput(e.target.value)}
                                            placeholder="مثال: 1, 3, 5-8, 12"
                                            className="w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-sky-500 focus:border-sky-500"
                                            disabled={isProcessing}
                                        />
                                        <p className="text-xs text-slate-500 mt-1">أدخل أرقام الصفحات مفصولة بفاصلة، ويمكنك استخدام الشرطة لتحديد نطاق.</p>
                                    </div>

                                    <div>
                                        <label htmlFor="filename" className="block text-sm font-medium text-slate-700 mb-1">
                                            اسم الملف الجديد
                                        </label>
                                        <input
                                            type="text"
                                            id="filename"
                                            value={outputFileName}
                                            onChange={(e) => setOutputFileName(e.target.value)}
                                            className="w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-sky-500 focus:border-sky-500"
                                            disabled={isProcessing}
                                        />
                                    </div>

                                    <button
                                        onClick={handleExtract}
                                        disabled={isProcessing || !pageInput || !outputFileName}
                                        className="w-full flex items-center justify-center gap-2 bg-sky-600 text-white font-bold py-3 px-4 rounded-md hover:bg-sky-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-sky-500 disabled:bg-slate-400 disabled:cursor-not-allowed transition-colors"
                                    >
                                        {isProcessing ? (
                                            <React.Fragment>
                                                <SpinnerIcon className="w-5 h-5" />
                                                <span>جاري المعالجة...</span>
                                            </React.Fragment>
                                        ) : (
                                            <span>استخراج الصفحات</span>
                                        )}
                                    </button>
                                </div>
                            )}

                            {error && (
                                <div className="bg-red-100 border-l-4 border-red-500 text-red-700 p-4 rounded-md" role="alert">
                                    <p className="font-bold">خطأ</p>
                                    <p>{error}</p>
                                </div>
                            )}
                            
                            {processedPdfUrl && (
                                <div className="bg-green-100 border-l-4 border-green-500 text-green-800 p-4 rounded-md flex items-center justify-between">
                                    <div>
                                        <p className="font-bold">نجاح!</p>
                                        <p>ملفك الجديد جاهز للتحميل.</p>
                                    </div>
                                    <a
                                        href={processedPdfUrl}
                                        download={outputFileName.endsWith('.pdf') ? outputFileName : `${outputFileName}.pdf`}
                                        className="inline-flex items-center gap-2 bg-green-600 text-white font-bold py-2 px-4 rounded-md hover:bg-green-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-green-500 transition-colors"
                                    >
                                        <DownloadIcon className="w-5 h-5" />
                                        تحميل الملف
                                    </a>
                                </div>
                            )}
                        </main>
                         <footer className="text-center mt-8 text-sm text-slate-500">
                            <p>خصوصيتك محفوظة. يتم التعامل مع جميع الملفات على جهازك مباشرةً.</p>
                        </footer>
                    </div>
                </div>
            );
        };
        
        // From index.tsx
        const rootElement = document.getElementById('root');
        if (!rootElement) {
          throw new Error("Could not find root element to mount to");
        }
        const root = ReactDOM.createRoot(rootElement);
        root.render(
          <React.StrictMode>
            <App />
          </React.StrictMode>
        );

    </script>
</body>
</html>
